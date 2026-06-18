---
type: research
date: 2026-04-30
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [TimescaleDB, pgvectorscale, prototype, docker, setup]
---

# TimescaleDB + pgvectorscale 로컬 프로토타입 구성

> Docker Compose 기반 로컬 개발 환경 설정.
> TimescaleDB + pgvectorscale 단일 인스턴스 구성 → 시계열 + 벡터 단일 SQL 검증용.

---

## 0. 현재 적재 현황 (Gate A · 2026-04-30)

| 테이블 | 행 수 | 비고 |
|--------|------|------|
| `team_state` | **106,566행** | 2,000경기 / win_prob + model_signal 적재 완료 |
| `teamfight_events` | **8,711건** | CHAMPION_KILL 클러스터링 결과 |
| `canonical_games` | **541경기** | |
| `composition_embeddings` | **1,572 nodes** | PCA 64dim / diskann 인덱스 ⚫ 미빌드 |
| `player_state` | 적재 미사용 | Gate A는 team_state 중심 파이프라인 운용 |
| `state_embedding` (128dim) | ⚫ 미구현 | Gate B 예정 |

> **컨테이너**: `lol-ontology-db` (Docker), `localhost:5432`, `loluser/lolpass/lol_ontology`

---

## 전제 조건

- Docker Desktop 설치
- Python 3.11+ (학습 스크립트용)
- psql 클라이언트 (쿼리 테스트용)

---

## 1. Docker Compose 구성

```yaml
# docker-compose.yml
version: '3.9'

services:
  timescaledb:
    image: timescale/timescaledb-ha:pg16-latest   # pgvectorscale 포함 버전
    container_name: lol-ontology-db
    environment:
      POSTGRES_USER: loluser
      POSTGRES_PASSWORD: lolpass
      POSTGRES_DB: lol_ontology
    ports:
      - "5432:5432"
    volumes:
      - timescale_data:/var/lib/postgresql/data
      - ./sql/init:/docker-entrypoint-initdb.d    # 초기화 SQL 자동 실행
    command: >
      postgres
        -c shared_preload_libraries='timescaledb,vectorscale'
        -c max_connections=200
        -c shared_buffers=512MB
        -c work_mem=32MB

volumes:
  timescale_data:
```

> **주의**: `timescale/timescaledb-ha` 이미지에 pgvectorscale이 번들 포함됨.
> 별도 빌드 불필요.

---

## 2. 초기화 SQL

아래 예시는 설계 설명용이 아니라, 현재 저장소의 `scraper/sql/init/02_tables.sql`과 최대한 맞춘 로컬 프로토타입 기준으로 읽는 것이 맞다.
최근 구현 기준 핵심 차이:

- `player_state`, `game_events`, `team_state`, `teamfight_events`, `composition_embeddings`가 실제 초기화 대상
- 보조 소스 오염 방지를 위해 `staging`, `ops` schema를 함께 생성
- 좌표계는 `pos_y`가 아니라 `pos_z`
- `team_state`는 실시간 예측 적재를 위해 `win_prob`, `model_signal` 컬럼을 사용
- 로컬 검증은 Match-v5 timeline 백필 + mock replay를 함께 사용

```sql
-- sql/init/01_extensions.sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS vectorscale CASCADE;  -- pgvector도 함께 설치됨

-- sql/init/02_tables.sql

-- ① player_state: timeline / live state 적재
--    좌표계: Live Stats API는 {x, z} 사용 (pos_y ❌ → pos_z ✅)
CREATE TABLE player_state (
  time              TIMESTAMPTZ     NOT NULL,
  match_id          TEXT            NOT NULL,
  participant_id    SMALLINT        NOT NULL,
  champion_id       SMALLINT,
  team_id           SMALLINT,
  pos_x             INT,
  pos_z             INT,
  alive             BOOLEAN,
  health            REAL,
  max_health        REAL,
  resource          REAL,
  max_resource      REAL,
  level             SMALLINT,
  total_gold        INT,
  cs                INT,
  q_cd              REAL,
  w_cd              REAL,
  e_cd              REAL,
  r_cd              REAL,
  flash_cd          REAL,
  state_embedding   vector(128),
  PRIMARY KEY (time, participant_id, match_id)
);

-- ② game_events: timeline events
CREATE TABLE game_events (
  time              TIMESTAMPTZ     NOT NULL,
  match_id          TEXT            NOT NULL,
  event_type        TEXT            NOT NULL,
  source_id         SMALLINT,
  target_id         SMALLINT,
  pos_x             INT,
  pos_z             INT,
  objective_type    TEXT,
  item_id           INT,
  gold_value        INT,
  PRIMARY KEY (time, match_id, event_type)
);

-- ③ team_state: 팀 단위 상태 / 실시간 예측 시계열
CREATE TABLE team_state (
  time              TIMESTAMPTZ     NOT NULL,
  match_id          TEXT            NOT NULL,
  team_id           SMALLINT        NOT NULL,
  total_gold        INT,
  kills             SMALLINT,
  towers            SMALLINT,
  dragons           SMALLINT,
  barons            SMALLINT,
  inhibitors        SMALLINT,
  cs                INT,
  win_prob          REAL,
  model_signal      REAL,
  PRIMARY KEY (time, match_id, team_id)
);

-- ④ teamfight_events: 한타 감지 결과 저장
CREATE TABLE teamfight_events (
  time              TIMESTAMPTZ     NOT NULL,
  match_id          TEXT            NOT NULL,
  fight_id          TEXT            NOT NULL,
  phase             TEXT            NOT NULL,
  centroid_x        REAL,
  centroid_z        REAL,
  participants      JSONB,
  duration_sec      SMALLINT,
  gold_swing        INT,
  kills_in_fight    SMALLINT,
  objective_secured TEXT,
  pre_state         vector(128),
  outcome           TEXT,
  PRIMARY KEY (time, fight_id)
);

-- ⑤ composition_embeddings: 챔피언 구성 벡터
CREATE TABLE composition_embeddings (
  comp_id           TEXT            PRIMARY KEY,
  team_id           SMALLINT,
  match_id          TEXT,
  game_time_sec     INT,
  comp_vector       vector(64),
  win               BOOLEAN,
  fight_occurred    BOOLEAN
);

```

하이퍼테이블 / 인덱스는 현재 `sql/init/03_hypertables.sql` 기준으로 다음처럼 잡혀 있다.

```sql
SELECT create_hypertable('player_state', 'time', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('game_events', 'time', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('team_state', 'time', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('teamfight_events', 'time', chunk_time_interval => INTERVAL '7 days');

CREATE INDEX ON composition_embeddings USING diskann (comp_vector);
CREATE INDEX ON teamfight_events USING diskann (pre_state) WHERE pre_state IS NOT NULL;

CREATE INDEX ON player_state (match_id, participant_id, time DESC);
CREATE INDEX ON game_events (match_id, event_type, time DESC);
CREATE INDEX ON teamfight_events (match_id, fight_id);
CREATE INDEX ON player_state (match_id, time DESC);

SELECT add_retention_policy('player_state', INTERVAL '90 days');
SELECT add_retention_policy('game_events', INTERVAL '90 days');
SELECT add_retention_policy('team_state', INTERVAL '90 days');
```

---

## 3. 실행 방법

```bash
# 컨테이너 시작
docker compose up -d

# 초기화 확인
docker exec -it lol-ontology-db psql -U loluser -d lol_ontology -c "\dx"

# 예상 출력:
#  timescaledb | 2.x.x | ...
#  vector      | 0.x.x | ...
#  vectorscale | 0.x.x | ...

# psql 접속
psql -h localhost -U loluser -d lol_ontology
```

기존 DB에 `canonical_games` 테이블만 추가하고 싶다면 migration SQL을 별도로 적용하면 된다.

```bash
docker exec -i lol-ontology-db \
  psql -U loluser -d lol_ontology \
  < scraper/sql/migrations/20260423_add_canonical_games.sql
```

---

## 4. 샘플 데이터 적재 / 운영 흐름

현재 로컬 프로토타입은 공식 소스 기반 두 경로를 함께 쓴다.

- 학습용 백필
  - `scripts/pipeline/collect_matches.py`
  - `scripts/pipeline/backfill_matches.py`
  - Match-v5 timeline을 `player_state`, `game_events`로 적재
- 실시간 추론 검증
  - `python -m scripts.realtime.live_stats_receiver <match_id>`
  - `python -m scripts.realtime.replay_cached_matches --limit N`
  - mock replay를 통해 `team_state.win_prob`, `team_state.model_signal` 적재
샘플 적재 예시는 현재 구현 기준으로 아래 형태가 맞다.

```python
# scripts/load_sample_data.py
import psycopg2
import requests
from datetime import datetime, timezone

# DB 연결
conn = psycopg2.connect(
    host="localhost", port=5432,
    dbname="lol_ontology",
    user="loluser", password="lolpass"
)
cur = conn.cursor()

RIOT_API_KEY = "RGAPI-xxxxxxxx"  # .env에서 로드
REGION = "asia"
MATCH_ID = "KR_7300000000"       # 테스트용 match ID

# Match-v5 timeline 가져오기
url = f"https://{REGION}.api.riotgames.com/lol/match/v5/matches/{MATCH_ID}/timeline"
headers = {"X-Riot-Token": RIOT_API_KEY}
resp = requests.get(url, headers=headers)
timeline = resp.json()

# participantFrames → player_state 적재
for frame in timeline["info"]["frames"]:
    ts = datetime.fromtimestamp(frame["timestamp"] / 1000, tz=timezone.utc)

    for pid_str, pf in frame["participantFrames"].items():
        pid = int(pid_str)
        cur.execute("""
            INSERT INTO player_state
              (time, match_id, participant_id, team_id, pos_x, pos_z,
               alive, level, total_gold, cs)
            VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)
            ON CONFLICT DO NOTHING
        """, (
            ts,
            MATCH_ID,
            pid,
            100 if pid <= 5 else 200,
            pf["position"]["x"],
            pf["position"].get("y"),
            True,
            pf["level"],
            pf["totalGold"],
            pf["minionsKilled"],
        ))

# timeline events → game_events 적재
ALLOWED = {
    "CHAMPION_KILL",
    "BUILDING_KILL",
    "ELITE_MONSTER_KILL",
    "WARD_PLACED",
    "WARD_KILL",
    "ITEM_PURCHASED",
}

for frame in timeline["info"]["frames"]:
    for event in frame.get("events", []):
        if event["type"] not in ALLOWED:
            continue
        ts_ev = datetime.fromtimestamp(event["timestamp"] / 1000, tz=timezone.utc)
        pos = event.get("position", {})
        cur.execute("""
            INSERT INTO game_events
              (time, match_id, event_type, source_id, target_id,
               pos_x, pos_z, objective_type, item_id, gold_value)
            VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)
            ON CONFLICT DO NOTHING
        """, (
            ts_ev,
            MATCH_ID,
            event["type"],
            event.get("killerId"),
            event.get("victimId"),
            pos.get("x"),
            pos.get("y"),
            event.get("buildingType") or event.get("monsterType"),
            event.get("itemId"),
            event.get("bounty", 0),
        ))

conn.commit()
cur.close()
conn.close()
print("✅ Sample data loaded")
```

---

## 5. 핵심 쿼리 검증

```sql
-- 검증 1: hypertable 파티셔닝 확인
SELECT show_chunks('player_state');

-- 검증 2: 특정 경기 타임라인 조회
SELECT time, participant_id, pos_x, pos_z, total_gold, level
FROM player_state
WHERE match_id = 'KR_7300000000'
ORDER BY time, participant_id;

-- 검증 3: 킬 이벤트만 필터
SELECT time, source_id AS killer,
       target_id AS victim,
       pos_x, pos_z
FROM game_events
WHERE match_id = 'KR_7300000000'
  AND event_type = 'CHAMPION_KILL'
ORDER BY time;

-- 검증 4: 골드 차이 추이 (TimescaleDB time_bucket 활용)
SELECT
  time_bucket('5 minutes', time) AS bucket,
  SUM(CASE WHEN team_id = 100 THEN total_gold ELSE 0 END) AS gold_blue,
  SUM(CASE WHEN team_id = 200 THEN total_gold ELSE 0 END) AS gold_red,
  SUM(CASE WHEN team_id = 100 THEN total_gold ELSE -total_gold END) AS gold_diff
FROM player_state
WHERE match_id = 'KR_7300000000'
GROUP BY bucket
ORDER BY bucket;

-- 검증 5: team_state 예측 시계열 확인
SELECT time, team_id, total_gold, kills, cs, win_prob, model_signal
FROM team_state
WHERE match_id = 'KR_7508335096'
ORDER BY time, team_id
LIMIT 20;

-- 검증 6: replay 후 경기 수 증가 확인
SELECT count(*) AS rows_after, count(DISTINCT match_id) AS matches_after
FROM team_state;

-- 검증 7: canonical game spine 확인
SELECT season, count(*)
FROM canonical_games
GROUP BY season
ORDER BY season;
```

---

## 5A. 공식 소스 운영 규칙

- `public` schema에는 Riot / lolesports raw와 raw 기반 derived만 적재한다.
- `canonical_games`는 `series_raw.json`, `game_windows.json`, `game_results.json` 같은 공식 캐시에서만 구성한다.
- 모델 학습 파이프라인은 공식 raw + raw 기반 derived만 읽는다.
- 비공식 보조 소스는 현재 로컬 프로토타입 운영 경로에서 사용하지 않는다.

---

## 6. pgvectorscale 임베딩 검증 (더미 데이터)

```python
# scripts/test_vector_search.py
import psycopg2
import numpy as np

conn = psycopg2.connect(
    host="localhost", port=5432,
    dbname="lol_ontology", user="loluser", password="lolpass"
)
cur = conn.cursor()

# 더미 composition embedding 삽입
for i in range(100):
    vec = np.random.rand(64).astype(np.float32).tolist()
    cur.execute("""
        INSERT INTO composition_embeddings
          (comp_id, match_id, team_id, game_time_sec, comp_vector, win, fight_occurred)
        VALUES (%s,%s,%s,%s,%s::vector,%s,%s)
    """, (
        f"comp_{i}", f"KR_000{i}", 100, 900 + i * 60,
        str(vec), bool(i % 2), bool(i % 3)
    ))

conn.commit()

# K-NN 검색 테스트 (StreamingDiskANN)
query_vec = np.random.rand(64).astype(np.float32).tolist()
cur.execute("""
    SELECT comp_id, match_id, win,
           comp_vector <=> %s::vector AS distance
    FROM composition_embeddings
    ORDER BY distance ASC
    LIMIT 5
""", (str(query_vec),))

results = cur.fetchall()
for row in results:
    print(f"comp_id={row[0]}, match={row[1]}, win={row[2]}, dist={row[3]:.4f}")

cur.close()
conn.close()
print("✅ Vector search test complete")
```

---

## 7. 다음 단계 (프로토타입 → 실험)

> **DDL 변경 요약 (2026-03-23)**: Live Stats API 데이터 샘플 분석 결과 반영
> - `pos_y` → `pos_z` 전면 교체 (Live Stats API 좌표계: `{x, z}`)
> - `player_state`: `health`, `health_max`, `health_pct`(GENERATED), `primary_resource`, `alive`, `respawn_timer`, 쿨타임 컬럼 6개 추가
> - `game_events`: `dragon_type`, `fight_duration`, `death_recap`, `max_cooldown`, `charges_remaining`, `is_interrupted` 추가
> - `teamfight_events`: `centroid_y` → `centroid_z`

- [x] Match-v5 timeline 수집 및 적재 ✅ (2,000경기 완료)
- [x] 한타 감지: CHAMPION_KILL 클러스터링 스크립트 ✅ (8,711건)
- [x] 한타 감지: 실시간 proximity 감지 구현 ✅ (mock 모드, Gate B에서 실 연결)
- [x] Live Stats API 수신기 구현 ✅ (`LiveStatsReceiver` mock_mode — Gate B에서 실 WebSocket 연결)
- [x] composition_embeddings 적재 ✅ (1,572 nodes / PCA 64dim)
- [ ] `state_embedding` 128차원 벡터 생성 (player_state) — **Gate B**
- [ ] diskann 인덱스 빌드 (composition_embeddings, teamfight_events.pre_state) — **Gate B**
- [ ] TimescaleDB continuous aggregate (5분 단위 골드 차이) — Gate B
- [ ] 관련 파일: [[live-stats-api-analysis]] — Live Stats API 완전 분석
