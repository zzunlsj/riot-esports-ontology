---
type: research
date: 2026-04-30
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [pipeline, match-v5, teamfight, detection, clustering, LightGBM]
---

# 학습 데이터 파이프라인 + 한타 감지 재설계

> API 검증 결과 반영: 실시간 HP/위치 대신 Match-v5 timeline(1분 간격) 기반으로 전환.
> 한타 감지 방식: rule-based 집결 감지 → **CHAMPION_KILL 이벤트 클러스터링**으로 재설계.

---

## 0. 현재 구현 상태 (Gate A · 2026-04-30)

| 컴포넌트 | 상태 | 실측 수치 |
|---------|------|---------|
| Match-v5 timeline 수집 | ✅ 완료 | **2,000경기** 캐시 (`data/timeline_cache/`) |
| `team_state` TimescaleDB 적재 | ✅ 완료 | **106,566행** (win_prob + model_signal) |
| `teamfight_events` 적재 | ✅ 완료 | **8,711건** (CHAMPION_KILL 클러스터링) |
| `canonical_games` | ✅ 완료 | **541경기** |
| `game_state` 임베딩 (128dim) | ⚫ 미구현 | Gate B 예정 |
| `diskann` 인덱스 빌드 | ⚫ 미구현 | Gate B 예정 |
| 실시간 적재 경로 | ✅ 완료 | `replay_cached_matches.py` → `LiveStatsReceiver` → DB |

> **실제 적재 경로**: `*_timeline.json` (캐시) → `LiveStatsReceiver.run_mock()` → `team_state` + `teamfight_events`
> 설계 문서의 `player_state` 직접 적재는 현재 미사용. 실시간 파이프라인이 `team_state` 중심으로 구현됨.

---

## 1. Match-v5 Timeline 기반 데이터 파이프라인

### 수집 대상

| 소스 | 수집 내용 | 비고 |
|------|---------|------|
| Match-v5 `/matches/{id}` | 경기 메타 (챔피언·룬·결과) | spectator-v5 대체 |
| Match-v5 `/matches/{id}/timeline` | 1분 프레임 + 이벤트 스트림 | 핵심 학습 데이터 |
| Challenger/GM ladder | 고급 경기 puuid 수집 | 데이터 품질 확보 |

### 수집 목표량

| 구분 | 목표 | 예상 한타 수 |
|------|------|------------|
| Phase 0 POC | 500경기 | ~2,500건 |
| Phase 1 실험 | 10,000경기 | ~50,000건 |
| Phase 2 운용 | 100,000경기 | ~500,000건 |

### 파이프라인 코드

```python
# scripts/pipeline/collect_matches.py
"""
Challenger/GM 사다리에서 match_id 수집 → timeline 적재
Rate limit: 20 req/sec, 100 req/2min (개발 키 기준)
Production 키: 500 req/10sec
"""
import time
import requests
import psycopg2
from psycopg2.extras import execute_batch
from dataclasses import dataclass
from datetime import datetime, timezone

API_KEY = "RGAPI-xxxx"
HEADERS = {"X-Riot-Token": API_KEY}
DB_CONN = "postgresql://loluser:lolpass@localhost:5432/lol_ontology"

QUEUE_ID = 420        # Ranked Solo/Duo
MIN_TIER = "CHALLENGER"


def get_challenger_puuids(region="kr") -> list[str]:
    """Challenger 리그 summoner IDs → puuids"""
    url = f"https://{region}.api.riotgames.com/lol/league/v4/challengerleagues/by-queue/RANKED_SOLO_5x5"
    resp = requests.get(url, headers=HEADERS).json()
    summoner_ids = [e["summonerId"] for e in resp["entries"][:50]]

    puuids = []
    for sid in summoner_ids:
        url = f"https://{region}.api.riotgames.com/lol/summoner/v4/summoners/{sid}"
        puuid = requests.get(url, headers=HEADERS).json()["puuid"]
        puuids.append(puuid)
        time.sleep(0.06)  # rate limit 준수
    return puuids


def get_match_ids(puuid: str, region="asia", count=20) -> list[str]:
    url = (f"https://{region}.api.riotgames.com/lol/match/v5/matches"
           f"/by-puuid/{puuid}/ids?queue={QUEUE_ID}&count={count}")
    return requests.get(url, headers=HEADERS).json()


def fetch_and_store_timeline(match_id: str, region="asia", conn):
    """Timeline 가져와서 player_state + game_events 적재"""
    url = f"https://{region}.api.riotgames.com/lol/match/v5/matches/{match_id}/timeline"
    data = requests.get(url, headers=HEADERS).json()

    if "info" not in data:
        return False

    frames = data["info"]["frames"]
    player_state_rows = []
    event_rows = []

    for frame in frames:
        ts = datetime.fromtimestamp(frame["timestamp"] / 1000, tz=timezone.utc)

        # participantFrames → player_state
        for pid_str, pf in frame["participantFrames"].items():
            player_state_rows.append((
                ts, match_id, pf["participantId"],
                pf["position"]["x"], pf["position"]["y"],
                pf["totalGold"], pf["currentGold"],
                pf["level"], pf["xp"],
                pf["minionsKilled"], pf["jungleMinionsKilled"]
            ))

        # events → game_events
        for event in frame.get("events", []):
            ev_ts = datetime.fromtimestamp(event["timestamp"] / 1000, tz=timezone.utc)
            pos = event.get("position", {})
            assisting = event.get("assistingParticipantIds", []) or []
            event_rows.append((
                ev_ts, match_id, event["type"], event["timestamp"],
                event.get("killerId") or event.get("creatorId") or event.get("participantId"),
                event.get("victimId"),
                pos.get("x"), pos.get("y"),
                event.get("itemId"), event.get("skillSlot"),
                event.get("wardType"), event.get("buildingType"),
                event.get("monsterType"), event.get("killType"),
                assisting if assisting else []
            ))

    cur = conn.cursor()
    execute_batch(cur, """
        INSERT INTO player_state
          (time, match_id, participant_id, pos_x, pos_y,
           total_gold, current_gold, level, xp, minions_killed, jungle_cs)
        VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)
        ON CONFLICT DO NOTHING
    """, player_state_rows, page_size=500)

    execute_batch(cur, """
        INSERT INTO game_events
          (time, match_id, event_type, timestamp_ms,
           source_participant_id, target_participant_id,
           pos_x, pos_y, item_id, skill_slot,
           ward_type, building_type, monster_type, kill_type, assisting_ids)
        VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)
        ON CONFLICT DO NOTHING
    """, event_rows, page_size=500)

    conn.commit()
    return True
```

---

## 2. 한타 감지 재설계: CHAMPION_KILL 클러스터링

### 재설계 이유

- **기존 설계**: 실시간 좌표 집결 감지 (HP%, 위치, 플래시 보유 등) → 공개 API 미지원
- **수정 설계**: 과거 경기 CHAMPION_KILL 이벤트의 시공간 클러스터링 → 한타 구간 재구성

### 알고리즘

```python
# scripts/pipeline/detect_teamfights.py
"""
CHAMPION_KILL 이벤트 클러스터링으로 한타 구간 탐지.

기준:
- 시간 클러스터: 30초 윈도우 내 킬 이벤트 3건 이상
- 공간 클러스터: 킬 이벤트 좌표 간 거리 < 2000 유닛
"""
import uuid
import numpy as np
import psycopg2
from datetime import timedelta

KILL_WINDOW_SEC = 30      # 클러스터링 시간 윈도우
SPATIAL_THRESHOLD = 2000  # 공간 거리 임계값 (유닛)
MIN_KILLS = 3             # 한타 판정 최소 킬 수
MIN_PARTICIPANTS = 4      # 최소 참가자 수 (양팀 합산)


def detect_teamfights_for_match(match_id: str, conn):
    cur = conn.cursor()

    # 킬 이벤트 조회
    cur.execute("""
        SELECT time, timestamp_ms,
               source_participant_id AS killer,
               target_participant_id AS victim,
               assisting_ids,
               pos_x, pos_y
        FROM game_events
        WHERE match_id = %s
          AND event_type = 'CHAMPION_KILL'
          AND pos_x IS NOT NULL
        ORDER BY time
    """, (match_id,))
    kills = cur.fetchall()

    if len(kills) < MIN_KILLS:
        return []

    fights = []
    used = set()

    for i, kill in enumerate(kills):
        if i in used:
            continue

        # 시간 윈도우 내 인접 킬 수집
        cluster = [i]
        base_ts = kill[1]  # timestamp_ms
        base_pos = np.array([kill[5], kill[6]])

        for j, other in enumerate(kills[i+1:], start=i+1):
            if j in used:
                continue
            time_diff_ms = other[1] - base_ts
            if time_diff_ms > KILL_WINDOW_SEC * 1000:
                break  # 시간순 정렬이므로 이후는 볼 필요 없음

            other_pos = np.array([other[5], other[6]])
            dist = np.linalg.norm(base_pos - other_pos)
            if dist < SPATIAL_THRESHOLD:
                cluster.append(j)

        if len(cluster) < MIN_KILLS:
            continue

        # 참가자 수 계산
        all_participants = set()
        for idx in cluster:
            k = kills[idx]
            if k[2]: all_participants.add(('k', k[2]))  # killer
            if k[3]: all_participants.add(('v', k[3]))  # victim
            for aid in (k[4] or []):
                all_participants.add(('a', aid))

        if len(all_participants) < MIN_PARTICIPANTS:
            continue

        # 한타 메타 계산
        cluster_kills = [kills[idx] for idx in cluster]
        coords = np.array([[k[5], k[6]] for k in cluster_kills])
        centroid = coords.mean(axis=0)

        start_ts = cluster_kills[0][1]
        end_ts = cluster_kills[-1][1]
        duration_sec = max((end_ts - start_ts) // 1000, 1)

        # 참가자 팀 분리 (participant_id 1-5 = 팀 100, 6-10 = 팀 200)
        team_a = [p[1] for p in all_participants if p[1] and p[1] <= 5]
        team_b = [p[1] for p in all_participants if p[1] and p[1] > 5]

        fight = {
            "fight_id": str(uuid.uuid4()),
            "match_id": match_id,
            "start_timestamp_ms": start_ts,
            "centroid_x": int(centroid[0]),
            "centroid_y": int(centroid[1]),
            "kills_in_fight": len(cluster),
            "duration_sec": duration_sec,
            "team_a_participants": team_a,
            "team_b_participants": team_b,
            "total_participants": len(all_participants),
        }
        fights.append(fight)

        for idx in cluster:
            used.add(idx)

    return fights


def enrich_with_gold_swing(fight: dict, match_id: str, conn) -> dict:
    """한타 전후 골드 차이 변화 = gold_swing"""
    cur = conn.cursor()
    ts_before = fight["start_timestamp_ms"] / 1000 - 60  # 1분 전
    ts_after = fight["start_timestamp_ms"] / 1000 + fight["duration_sec"] + 30

    cur.execute("""
        SELECT team_id,
               SUM(total_gold) as gold
        FROM player_state
        WHERE match_id = %s
          AND EXTRACT(EPOCH FROM time) BETWEEN %s AND %s
        GROUP BY team_id
        ORDER BY team_id
    """, (match_id, ts_before, ts_before + 10))
    before = {r[0]: r[1] for r in cur.fetchall()}

    cur.execute("""
        SELECT team_id,
               SUM(total_gold) as gold
        FROM player_state
        WHERE match_id = %s
          AND EXTRACT(EPOCH FROM time) BETWEEN %s AND %s
        GROUP BY team_id
        ORDER BY team_id
    """, (match_id, ts_after, ts_after + 10))
    after = {r[0]: r[1] for r in cur.fetchall()}

    gold_before = (before.get(100, 0) or 0) - (before.get(200, 0) or 0)
    gold_after = (after.get(100, 0) or 0) - (after.get(200, 0) or 0)
    fight["gold_swing"] = gold_after - gold_before

    return fight
```

---

## 3. 한타 레이블링 (지도학습용)

```python
def label_teamfight_outcome(fight: dict, match_id: str, conn) -> dict:
    """
    한타 결과 레이블:
    - 킬 수 비교 (victim팀 기준)
    - 이후 오브젝트 획득 여부
    """
    # 킬된 선수 팀 집계
    team_a_deaths = sum(
        1 for k in kills_in_fight
        if k["victim_id"] and k["victim_id"] <= 5
    )
    team_b_deaths = sum(
        1 for k in kills_in_fight
        if k["victim_id"] and k["victim_id"] > 5
    )

    if team_a_deaths > team_b_deaths:
        fight["outcome"] = "team_b_win"
    elif team_b_deaths > team_a_deaths:
        fight["outcome"] = "team_a_win"
    else:
        fight["outcome"] = "draw"

    # 한타 직후 30초 내 오브젝트 획득
    cur = conn.cursor()
    ts_fight_end = fight["start_timestamp_ms"] + fight["duration_sec"] * 1000
    cur.execute("""
        SELECT monster_type, building_type, source_participant_id
        FROM game_events
        WHERE match_id = %s
          AND timestamp_ms BETWEEN %s AND %s
          AND event_type IN ('ELITE_MONSTER_KILL', 'BUILDING_KILL')
    """, (match_id, ts_fight_end, ts_fight_end + 30000))
    objs = cur.fetchall()
    fight["objective_converted"] = [r[0] or r[1] for r in objs] if objs else []

    return fight
```

---

## 4. 데이터셋 통계

| 지표 | Phase 0 목표 | **Gate A 실측** | Phase 1 목표 |
|------|------------|--------------|------------|
| 수집 경기 수 | 500 | **2,000** | 10,000 |
| `team_state` 행 수 | - | **106,566행** | - |
| 한타 이벤트 수 | 2,500 | **8,711건** | 50,000 |
| 한타당 평균 킬 수 | 3~8 | ~4.4 (추정) | - |
| 팀A 승 비율 | ~50% | ~50% | ~50% |
| 평균 한타 지속 시간 | 10~25초 | - | - |
| 한타 당 평균 gold_swing | ±2,000g | - | - |

---

## 5. 파이프라인 전체 흐름

```
① Challenger 사다리 소환사 ID 수집 (weekly)
        ↓
② puuid → match_id 목록 수집 (20경기/인)
        ↓
③ match-v5 timeline JSON 캐시 저장 (data/timeline_cache/*_timeline.json)
        ↓  ✅ Gate A 완료 (2,000경기)
④ replay_cached_matches.py → LiveStatsReceiver.run_mock()
   → team_state (106,566행) + teamfight_events (8,711건) 적재
        ↓  ✅ Gate A 완료
⑤ 골드 스윙 + 오브젝트 레이블 추가
        ↓  ✅ Gate A 완료
⑥ 구성 임베딩 생성 (composition_embedding → composition_embeddings 1,572 nodes)
        ↓  ✅ Gate A 완료 (PCA 64dim)
⑦ game_state 임베딩 생성 (128차원 → team_state.state_embedding)
        ↓  ⚫ Gate B 미구현
⑧ diskann 인덱스 빌드 (pgvectorscale)
        ↓  ⚫ Gate B 미구현
⑨ XGBoost / LightGBM 학습 (Win Probability — EXP004 AUC 0.8353)
           ✅ Gate A 완료
```

---

## 6. 관련 파일

- [[api-verification]] — 데이터 소스 전략 (TRACK A)
- [[local-prototype-setup]] — DB 테이블 DDL
- [[champion-embedding-design]] — 구성 임베딩 설계
- [[_index]] — 프로젝트 전체 개요
