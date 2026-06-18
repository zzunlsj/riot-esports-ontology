---
type: research
date: 2026-04-30
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [API, spectator, match-v5, data-source, verification]
---

# Riot API 데이터 구조 검증

> 프로젝트에서 가정한 데이터 가용성을 실제 API 스펙과 대조 검증한 결과물.
> **최초 발견 (2026-03-23 오전)**: 공개 spectator API만으로는 실시간 HP/마나/쿨타임/위치 수집 불가능 → TRACK A/B 2트랙 전략 수립.
> **수정 발견 (2026-03-23 오후)**: 실제 수신 데이터 샘플 분석 결과 **Riot Esports Live Stats API**가 1초 단위 전체 플레이어 상태를 제공함을 확인 → 원래 아키텍처 설계 완전 유효.
> → 상세 분석: [[live-stats-api-analysis]]

---

## 1. spectator-v5 (공개 API)

> v4에서 v5로 업데이트됨. `puuid`, `summonerId`, `riotId` 필드 추가.

### CurrentGameInfoDTO

| 필드 | 타입 | 설명 |
|------|------|------|
| `gameId` | number | 게임 고유 ID |
| `gameType` | string | 게임 타입 (MATCHED_GAME 등) |
| `gameMode` | string | CLASSIC, ARAM 등 |
| `isCustomGame` | boolean | 커스텀 게임 여부 |
| `gameStartTime` | number | Unix timestamp (ms) |
| `gameLength` | number | **게임 경과 시간 (초)** |
| `platformId` | string | KR, NA1 등 서버 |
| `mapId` | number | 맵 ID (11 = SR) |
| `gameQueueConfigId` | number | 큐 타입 ID |
| `bannedChampions` | BannedChampionDTO[] | 밴 챔피언 목록 |
| `observers` | ObserverDTO | 관전 암호화 키 |
| `participants` | CurrentGameParticipantDTO[] | 참가자 10명 |

### CurrentGameParticipantDTO

| 필드 | 타입 | 설명 |
|------|------|------|
| `puuid` | string | 플레이어 고유 ID (78자) |
| `summonerId` | string | 소환사 ID (63자) |
| `riotId` | string | "소환사명#지역" 형식 |
| `summonerName` | string | 소환사 이름 |
| `championId` | number | 챔피언 ID |
| `teamId` | number | 100 (블루) / 200 (레드) |
| `spell1Id` | number | 소환사 주문 1 |
| `spell2Id` | number | 소환사 주문 2 |
| `profileIconId` | number | 프로필 아이콘 |
| `bot` | boolean | 봇 여부 |
| `perks` | PerksDTO | 룬 구성 |
| `gameCustomizationObjects` | array | 추가 커스텀 데이터 |

### PerksDTO

```json
{
  "perkIds": [8128, 8143, 8136, 8135, 8304, 8345],
  "perkStyle": 8100,
  "perkSubStyle": 8300
}
```

### ⚠️ spectator-v5로 수집 불가능한 데이터

| 데이터 | 가용성 | 비고 |
|--------|--------|------|
| 실시간 위치 (pos_x, pos_y) | ❌ 불가 | 공개 API에 미포함 |
| 현재 HP / HP% | ❌ 불가 | 미포함 |
| 현재 마나 / 마나% | ❌ 불가 | 미포함 |
| 스킬 쿨타임 (Q/W/E/R) | ❌ 불가 | 미포함 |
| 플래시 쿨타임 | ❌ 불가 | 미포함 |
| 현재 골드 | ❌ 불가 | 미포함 |
| 현재 아이템 목록 | ❌ 불가 | 미포함 |
| KDA / CS (진행 중) | ❌ 불가 | 명시적으로 미제공 |
| 현재 버프/디버프 | ❌ 불가 | 미포함 |

---

## 2. Live Client Data API (localhost:2999)

로컬 클라이언트에서만 접근 가능. **단일 플레이어(자기 자신)의 데이터만 제공.**

### /liveclientdata/allgamedata 주요 필드

**activePlayer (나 자신)**:
```json
{
  "championStats": {
    "abilityHaste": 15,
    "abilityPower": 0,
    "armor": 74,
    "attackDamage": 79,
    "attackSpeed": 0.72,
    "cooldownReduction": -0.0,
    "currentHealth": 1840,
    "maxHealth": 1840,
    "resourceMax": 332,
    "resourceValue": 332,
    "resourceType": "MANA"
  },
  "abilities": {
    "Q": { "abilityLevel": 5, "isActive": false, "cooldown": 0 },
    "W": { "abilityLevel": 5, "isActive": false, "cooldown": 8.2 },
    "E": { "abilityLevel": 5, "isActive": false, "cooldown": 0 },
    "R": { "abilityLevel": 3, "isActive": false, "cooldown": 45.1 }
  }
}
```

**allPlayers (모든 참가자 - 제한적)**:
```json
{
  "summonerName": "...",
  "championName": "...",
  "isBot": false,
  "isDead": false,
  "respawnTimer": 0,
  "scores": {
    "assists": 3,
    "creepScore": 87,
    "deaths": 1,
    "kills": 4,
    "wardScore": 12
  },
  "items": [...],
  "summonerSpells": { "summonerSpellOne": {...}, "summonerSpellTwo": {...} }
}
```

### ⚠️ Live Client API 한계

- 자신의 HP/마나/쿨타임은 정확히 제공
- 다른 플레이어의 위치·HP·쿨타임은 **제공하지 않음** (Riot 정책)
- 로컬 클라이언트 전용 → **서버사이드 집계 불가**
- 스펙테이터 시스템에 직접 활용 불가능

---

## 3. Match-v5 Timeline (과거 경기 학습용)

> 공개 API 중 **가장 풍부한 게임 상태 데이터**. 단, **경기 종료 후** 제공.

### participantFrames (1분 간격 스냅샷)

```json
{
  "1": {
    "participantId": 1,
    "position": { "x": 512, "y": 512 },
    "currentGold": 1250,
    "totalGold": 4820,
    "level": 9,
    "xp": 5820,
    "minionsKilled": 78,
    "jungleMinionsKilled": 14,
    "dominionScore": 0,
    "teamScore": 0
  }
}
```

### timeline event 타입 목록

| 이벤트 | 주요 필드 |
|--------|---------|
| `CHAMPION_KILL` | killerId, victimId, assistingParticipantIds[], position{x,y}, timestamp |
| `ITEM_PURCHASED` | participantId, itemId, timestamp |
| `ITEM_SOLD` | participantId, itemId, timestamp |
| `ITEM_UNDO` | participantId, beforeId, afterId |
| `ITEM_DESTROYED` | participantId, itemId |
| `SKILL_LEVEL_UP` | participantId, skillSlot(1-5), levelUpType |
| `LEVEL_UP` | participantId, level |
| `WARD_PLACED` | wardType, creatorId, position |
| `WARD_KILL` | wardType, killerId, position |
| `BUILDING_KILL` | buildingType, laneType, towerType, teamId, killerId, position |
| `ELITE_MONSTER_KILL` | monsterType(BARON_NASHOR/DRAGON/etc), killerId, position |
| `OBJECTIVE_BOUNTY_PRESTART` | teamId, actualStartTime |
| `CHAMPION_SPECIAL_KILL` | killType(KILL_FIRST_BLOOD/KILL_MULTI_KILL), multikillLength |
| `GAME_END` | winningTeam |

### Timeline 한계

- **1분 간격** 스냅샷 → 1초 단위 실시간 모니터링에 직접 사용 불가
- 한타 감지에 필요한 **초 단위 위치 데이터** 없음
- 학습 데이터셋 구성용으로는 최적

---

## 4. Riot Esports Data / GRID (프로 경기 전용)

> 공개 API가 아닌 **파트너십 기반** 데이터 플랫폼.

### 구조

```
[Riot Games] → [GRID (League Data Platform)] → [파트너사]
```

- **LDP (League Data Platform)**: Riot이 GRID와 공동 운영하는 중앙 데이터 허브
- **70개 이상 권리보유자**에게 실시간 데이터 공급
- 사용처: 미디어, 베팅, 선수 코칭, 무결성 모니터링

### GRID가 제공하는 프로 경기 데이터 (추정)

- 실시간 게임 프레임 (위치, HP, 골드, 쿨타임 포함)
- 경기 이벤트 스트림 (킬, 오브젝트, 와드 등)
- 선수/팀 메타데이터

### 접근 방법

- **B2B 파트너십** 필요 (grid.gg 통해 신청)
- Riot Esports 공식 데이터: riotesportsdata.com
- 직접 활용 불가 (개인/소규모 프로젝트 불가)

---

## 5. 아키텍처 영향 및 수정 방향

### 기존 가정 vs 실제 (최종 확인)

| 항목 | 기존 가정 | 1차 검증 결과 | 최종 확인 (Live Stats API) |
|------|---------|------|------|
| 1초 단위 HP/위치 | spectator-v5로 수집 | ❌ 공개 API 미제공 | ✅ **Live Stats API로 가능** |
| 스킬 쿨타임 추적 | Live Client API 활용 | ❌ 로컬 자신만 가능 | ✅ **Live Stats API ability1~4CooldownRemaining** |
| 프로 경기 실시간 | Riot Esports API | GRID 파트너십 필요 | ✅ **Live Stats API 직접 수신 가능** |
| 과거 경기 학습 | Match-v5 timeline | ✅ 가능 (1분 간격) | ✅ 동일 (학습 데이터 수집용) |
| 좌표계 | {x, y} | {x, y} 가정 | ⚠️ **{x, z}** — pos_y → pos_z 수정 필요 |

### 확정된 데이터 소스 전략 (2026-04-30 현황 반영)

```
TRACK A — 실시간 운용 (Live Stats API) ← Gate B 진행 중
  ├── LolesportsLivePoller (lolesports_live_poller.py) ✅ 구현 완료
  │   ├── window endpoint (경기 목록·상태) + details endpoint (프레임 상세) 병렬 폴링
  │   └── details_frame → LiveStatsReceiver에 전달 (HP%, 데미지 비율, 킬관여)
  ├── LiveStatsReceiver (live_stats_receiver.py) ✅ 구현 완료
  │   ├── on_frame() → WinProbPredictor → team_state TimescaleDB 적재
  │   ├── TeamfightDetector → 한타 구간 감지 + FightWindow 요약
  │   └── Bedrock commentary (mock 모드 운용 중, 실 크레덴셜 미연결)
  ├── WinProbPredictor (win_prob_predictor.py) ✅ 구현 완료
  │   ├── EXP004 .ubj (XGBoost) 우선 로드, 실패 시 EXP003 .pkl 폴백
  │   └── rich 피처: HP%, 데미지 비율, 와드, 킬관여 (details_frame 필요)
  ├── spectator-v5 → 경기 메타 (챔피언·룬·소환사주문) 보완
  └── Data Dragon → 정적 챔피언 데이터 (챔피언명, 스킬명 매핑)

TRACK B — 학습 데이터 (match-v5 timeline) ✅ Gate A 완료
  ├── Match-v5 /matches/{id}/timeline → 2,000경기 캐시 수집 완료
  │   (1분 간격 위치·골드·레벨 + 이벤트 스트림)
  ├── team_state TimescaleDB: 2,000경기 / 106,566행 적재 완료
  ├── CHAMPION_KILL 클러스터링 → TeamfightDetector, 8,711건 레이블링
  └── win_prob_swings.csv: Top 50 스윙 경기 추출 완료

  ※ GRID 파트너십은 필요 없어짐 (Live Stats API로 커버)
```

### player_state 테이블 최종안

```sql
-- Live Stats API 기준 전체 필드 포함
-- 상세 DDL: local-prototype-setup.md 참조
-- 핵심 변경: pos_y → pos_z, health/cooldown 컬럼 추가
CREATE TABLE player_state (
  time              TIMESTAMPTZ NOT NULL,
  match_id          TEXT        NOT NULL,
  participant_id    SMALLINT    NOT NULL,
  pos_x             INT,
  pos_z             INT,              -- ← 좌표계 수정 (z축)
  alive             BOOLEAN,
  health            SMALLINT,
  health_max        SMALLINT,
  health_pct        FLOAT4 GENERATED ALWAYS AS (health::float / NULLIF(health_max, 0)) STORED,
  primary_resource  SMALLINT,
  total_gold        INT,
  current_gold      INT,
  level             SMALLINT,
  q_cd              SMALLINT, w_cd SMALLINT, e_cd SMALLINT, r_cd SMALLINT,
  flash_cd          SMALLINT, tp_cd SMALLINT,
  state_embedding   vector(128)
);
```

---

## 6. 결론 및 권고사항 (최종)

**핵심 발견 3가지 (수정)**:

1. **spectator-v5는 메타 정보만 제공** — 챔피언·룬·소환사주문은 알 수 있으나 실시간 상태(HP/마나/위치/쿨타임)는 없음. 보조 소스로 활용.

2. **Live Stats API가 핵심 실시간 데이터 소스** — `stats_update` 이벤트로 1초 단위 전체 플레이어 상태(위치, HP, 쿨타임, 골드) 수신 가능. 원래 아키텍처 설계가 완전히 유효. 좌표계만 `{x, z}`로 수정 필요.

3. **학습 데이터는 match-v5 timeline, 실시간은 Live Stats API** — 투트랙 전략. GRID 파트너십 없이 POC부터 상용화까지 진행 가능.

**Gate A 완료 현황 (2026-04-29)**:
- [x] match-v5 timeline 기반 학습 데이터 파이프라인 설계 완료 → [[data-pipeline-and-teamfight-detection]]
- [x] DDL pos_y → pos_z 수정, health/cooldown 컬럼 추가 완료 → [[local-prototype-setup]]
- [x] Live Stats API 수신기 구현 완료 (LiveStatsReceiver + LolesportsLivePoller)
- [x] 2,000경기 수집 완료 → team_state 106,566행 적재
- [x] LightGBM Win Probability 실험 완료 (EXP004 AUC 0.8353) → [[win-probability-lightgbm]]
- [x] TeamfightDetector 구현, EXP-TF-001 XGBoost Acc 83.4% → [[data-pipeline-and-teamfight-detection]]

**Gate B 진행 중 (2026-04-30~)**:
- [ ] Bedrock 실 크레덴셜 연결 (현재 mock 모드)
- [ ] LolesportsLivePoller E2E 실시간 테스트 (실제 경기 연결)
- [ ] WebSocket 레이턴시 측정 (목표 < 2초)
- [ ] EXP-PG-001 픽/밴 승률 예측 모델
- [x] win_prob_swings.csv Top 50 갱신 완료 → [[gate-a-completion]]
