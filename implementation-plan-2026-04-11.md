---
type: project
date: 2026-04-11
project: Riot Esports 실시간 경기 맥락 온톨로지
status: completed
tags: [plan, implementation, LightGBM, Neo4j, TimescaleDB, pgvectorscale, Bedrock]
---

# Esports Ontology 구현 플랜 (2026-04-11)

> **✅ 실행 완료 — Gate A 통과 (2026-04-29)**
> 이 문서는 2026-04-11 작성된 구현 플랜입니다. 모든 Phase가 실행 완료됐으며 Gate A 기준을 충족했습니다.
> 다음 단계(Gate B)는 [[goal-metrics-roadmap]] 참조.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 수집된 LCK 2025 데이터를 기반으로 Neo4j 그래프 구축 → TimescaleDB 학습 데이터 적재 → Win Probability 모델 학습 → 실시간 Live Stats API 수신기 구현까지 End-to-End 파이프라인 완성

**Architecture:** Neo4j(정적 온톨로지) + TimescaleDB+pgvectorscale(동적·벡터) 이중 DB 구조. Match-v5 timeline으로 Win Probability LightGBM 학습 후, Live Stats API WebSocket 수신기가 1초 단위 상태를 수신해 실시간 예측 제공.

**Tech Stack:** Python 3.11, Neo4j Desktop (bolt), TimescaleDB (Docker), pgvectorscale, LightGBM, scikit-learn, asyncio/websockets, AWS Bedrock (Claude API)

---

## 현재 완료 상태 (Gate A 기준 · 2026-04-29)

- ✅ LCK 2025 데이터 수집 완료: 시리즈 209, 게임 685, PlayerGameStats 5,410개 (캐시에 저장)
- ✅ 챔피언 온톨로지 수집: 172개 챔피언, 패치변화 280건 (14.1~15.24)
- ✅ Neo4j 스키마 + 파이프라인 코드 완성 (`scraper/neo4j_client.py`, `run_pipeline.py`)
- ✅ TimescaleDB 로컬 Docker 구축 완료 (`lol-ontology-db`, timescaledb/vector/vectorscale 확인)
- ✅ Match-v5 timeline 백필 수집 구현 및 검증 완료 (`collect_matches.py`, `backfill_matches.py`)
- ✅ Win Probability 피처 재생성 및 모델 학습 완료 (`game_state_features.py`, `train_win_prob.py`)
- ✅ EXP003 AUC = 0.8424 → **EXP004 AUC = 0.8353** (LightGBM 16피처, GroupShuffleSplit)
- ✅ Live Stats mock replay 기반 실시간 예측/한타 감지/해설 파이프라인 구현 완료
- ✅ `team_state`에 `win_prob`, `model_signal` 실시간 적재 검증 완료 — **106,566행 / 2,000경기**
- ✅ 승률 급변 구간 분석 도구 추가 (`win_prob_swings.py`) — Top 50 CSV 생성
- ✅ 캐시 경기 일괄 mock 재생 배치 추가 (`replay_cached_matches.py`) — 전체 2,000경기 완주
- ✅ Bedrock 프롬프트 템플릿 설계 완료 (`bedrock-prompt-templates.md`)
- ✅ 챔피언 구성 임베딩 생성 완료 — Neo4j 1,572 노드 / PCA 64차원 / 분산 설명률 91.3%
- ✅ EXP-TF-001 Teamfight Outcome XGBoost 학습 완료 — Acc 83.4% / AUC 0.91 / 8,711건

**Gate B 미완료 항목:**
- ❌ Bedrock 실제 자격증명 연결 테스트 (NoCredentialsError, mock 모드 운용 중)
- ❌ Neo4j 온톨로지 관계 확장 (CHAINS_WITH 등 심화 관계) — Gate B P2
- ❌ game_state 임베딩(128dim) + diskann 인덱스 — Gate B
- ❌ EXP-PG-001 Pre-game K-NN — Gate B P2

---

## 파일 구조 (신규 생성 대상)

```
scraper/
├── scripts/
│   ├── pipeline/
│   │   ├── collect_matches.py       # Match-v5 Challenger 데이터 수집
│   │   └── label_teamfights.py      # 한타 이벤트 클러스터링 + 레이블
│   ├── features/
│   │   ├── game_state_features.py   # Win Prob 피처 엔지니어링
│   │   ├── win_prob_swings.py       # 승률 급변 구간 분석
│   │   └── champion_embeddings.py   # 구성 벡터 64차원 생성
│   ├── models/
│   │   ├── train_win_prob.py        # LightGBM 학습 (EXP-001~005)
│   │   └── win_prob_predictor.py    # 추론 클래스 (인터페이스 정의)
│   └── realtime/
│       ├── live_stats_receiver.py   # WebSocket 수신기 (asyncio)
│       ├── teamfight_detector.py    # 실시간 룰 기반 한타 감지
│       └── replay_cached_matches.py # 캐시 경기 일괄 mock 재생
├── sql/
│   ├── init/
│   │   ├── 01_extensions.sql        # timescaledb, vectorscale 활성화
│   │   ├── 02_tables.sql            # player_state, game_events 등
│   │   └── 03_hypertables.sql       # TimescaleDB 하이퍼테이블 변환
│   └── queries/
│       └── vector_search.sql        # pgvectorscale 유사도 검색 예시
├── docker-compose.yml               # TimescaleDB + pgvectorscale
├── models/                          # 학습된 모델 저장 (.pkl)
└── notebooks/
    └── win_prob_analysis.ipynb      # 실험 결과 분석
```

---

## Phase 1: Neo4j 데이터 적재 + 검증

### Task 1-1: Neo4j Desktop 기동 + 연결 확인

**Files:**
- Verify: `scraper/.env`
- Run: `scraper/test_and_verify.py`

- [ ] **Step 1: Neo4j Desktop에서 DB 기동**

Neo4j Desktop 앱을 열고 로컬 DB를 Start. DB가 실행 중인 상태에서:

```bash
cd 1-projects/riot-esports-ontology/scraper
source .venv/bin/activate  # 가상환경 없으면: python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt
```

- [ ] **Step 2: `.env` 파일 확인/생성**

```bash
# scraper/.env 내용 확인
cat .env
```

없으면 생성:
```
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=yourpassword
```

- [ ] **Step 3: 연결 테스트**

```bash
python test_and_verify.py
```

Expected: `Neo4j 연결 성공` 출력

---

### Task 1-2: 수집 데이터 Neo4j 적재

**Files:**
- Run: `scraper/run_pipeline.py`
- Verify: `scraper/neo4j_client.py`

- [ ] **Step 1: Dry-run으로 데이터 확인**

```bash
python run_pipeline.py --step roster --dry-run
```

Expected: 선수 목록 출력, 오류 없음

- [ ] **Step 2: 선수/팀 적재**

```bash
python run_pipeline.py --step neo4j
```

Expected: `Player N개, Team M개 업서트 완료`

- [ ] **Step 3: 경기 데이터 적재**

```bash
python run_pipeline.py --step games
python run_pipeline.py --step game-results
python run_pipeline.py --step player-stats
```

각 단계 완료 후 로그에서 에러 확인.

- [ ] **Step 4: 챔피언 온톨로지 적재**

```bash
python run_pipeline.py --step champions
```

- [ ] **Step 5: Neo4j Browser에서 검증 쿼리 실행**

Neo4j Browser (`http://localhost:7474`) 열고:

```cypher
// 전체 노드 수 확인
MATCH (n) RETURN labels(n) AS label, count(*) AS cnt ORDER BY cnt DESC;
```

Expected:
- Player: ~114명
- Team: ~10개
- Game: ~685개
- PlayerGameStats: ~5,410개
- Champion: ~172개

```cypher
// Faker 스탯 확인
MATCH (p:Player {riotId: "T1 Faker"})-[:HAD_STATS]->(s:PlayerGameStats)
RETURN avg(s.kills) AS avgK, avg(s.deaths) AS avgD, count(s) AS games
```

- [ ] **Step 6: 커밋**

```bash
# 소스코드 변경 없음 — 데이터 적재 확인 노트 작성
```

---

## Phase 2: TimescaleDB 로컬 프로토타입 구축

### Task 2-1: Docker Compose + 초기화 SQL 파일 생성

**Files:**
- Create: `scraper/docker-compose.yml`
- Create: `scraper/sql/init/01_extensions.sql`
- Create: `scraper/sql/init/02_tables.sql`
- Create: `scraper/sql/init/03_hypertables.sql`

- [ ] **Step 1: docker-compose.yml 생성**

```yaml
# scraper/docker-compose.yml
version: '3.9'

services:
  timescaledb:
    image: timescale/timescaledb-ha:pg16-latest
    container_name: lol-ontology-db
    environment:
      POSTGRES_USER: loluser
      POSTGRES_PASSWORD: lolpass
      POSTGRES_DB: lol_ontology
    ports:
      - "5432:5432"
    volumes:
      - timescale_data:/var/lib/postgresql/data
      - ./sql/init:/docker-entrypoint-initdb.d
    command: >
      postgres
        -c shared_preload_libraries='timescaledb,vectorscale'
        -c max_connections=200
        -c shared_buffers=512MB

volumes:
  timescale_data:
```

- [ ] **Step 2: 01_extensions.sql 생성**

```sql
-- scraper/sql/init/01_extensions.sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS vectorscale CASCADE;
```

- [ ] **Step 3: 02_tables.sql 생성**

```sql
-- scraper/sql/init/02_tables.sql

-- 좌표계 주의: Live Stats API는 {x, z}. pos_y 아님.
CREATE TABLE player_state (
  time              TIMESTAMPTZ     NOT NULL,
  match_id          TEXT            NOT NULL,
  participant_id    SMALLINT        NOT NULL,
  champion_id       SMALLINT,
  team_id           SMALLINT,           -- 100(blue) / 200(red)
  pos_x             INT,
  pos_z             INT,                -- Live Stats API: z축
  alive             BOOLEAN,
  health            REAL,               -- 현재 HP
  max_health        REAL,
  resource          REAL,               -- 마나/에너지
  max_resource      REAL,
  level             SMALLINT,
  total_gold        INT,
  cs                INT,
  q_cd              REAL,
  w_cd              REAL,
  e_cd              REAL,
  r_cd              REAL,
  flash_cd          REAL,
  state_embedding   vector(128)         -- pgvectorscale
);

CREATE TABLE game_events (
  time              TIMESTAMPTZ     NOT NULL,
  match_id          TEXT            NOT NULL,
  event_type        TEXT            NOT NULL,  -- KILL/OBJECTIVE/WARD/ITEM
  source_id         SMALLINT,
  target_id         SMALLINT,
  pos_x             INT,
  pos_z             INT,
  objective_type    TEXT,
  item_id           INT,
  gold_value        INT
);

CREATE TABLE team_state (
  time              TIMESTAMPTZ     NOT NULL,
  match_id          TEXT            NOT NULL,
  team_id           SMALLINT        NOT NULL,  -- 100 or 200
  total_gold        INT,
  kills             SMALLINT,
  towers            SMALLINT,
  dragons           SMALLINT,
  barons            SMALLINT,
  inhibitors        SMALLINT,
  cs                INT
);

CREATE TABLE teamfight_events (
  time              TIMESTAMPTZ     NOT NULL,
  match_id          TEXT            NOT NULL,
  fight_id          TEXT            NOT NULL,
  phase             TEXT            NOT NULL,  -- PRE/ACTIVE/POST
  centroid_x        REAL,
  centroid_z        REAL,
  participants      JSONB,
  duration_sec      SMALLINT,
  gold_swing        INT,
  kills_in_fight    SMALLINT,
  objective_secured TEXT,
  pre_state         vector(128),               -- 한타 직전 상태 벡터
  outcome           TEXT                        -- 'blue_win'/'red_win'/'draw'
);

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

- [ ] **Step 4: 03_hypertables.sql 생성**

```sql
-- scraper/sql/init/03_hypertables.sql

-- 시계열 테이블 → Hypertable 변환 (1일 청크)
SELECT create_hypertable('player_state', 'time', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('game_events', 'time', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('team_state', 'time', chunk_time_interval => INTERVAL '1 day');
SELECT create_hypertable('teamfight_events', 'time', chunk_time_interval => INTERVAL '7 days');

-- StreamingDiskANN 인덱스 (pgvectorscale)
CREATE INDEX ON composition_embeddings USING diskann (comp_vector);
CREATE INDEX ON teamfight_events USING diskann (pre_state) WHERE pre_state IS NOT NULL;

-- 기본 인덱스
CREATE INDEX ON player_state (match_id, participant_id, time DESC);
CREATE INDEX ON game_events (match_id, event_type, time DESC);
CREATE INDEX ON teamfight_events (match_id, fight_id);

-- player_state: 90일 보존 정책 (teamfight_events는 영구 보존)
SELECT add_retention_policy('player_state', INTERVAL '90 days');
```

- [ ] **Step 5: Docker 기동 + 연결 확인**

```bash
cd scraper/
docker-compose up -d
sleep 5
psql "postgresql://loluser:lolpass@localhost:5432/lol_ontology" -c "\dt"
```

Expected: player_state, game_events, team_state, teamfight_events, composition_embeddings 테이블 목록 출력

- [ ] **Step 6: 커밋**

```bash
git add docker-compose.yml sql/
git commit -m "feat: add TimescaleDB+pgvectorscale docker setup with DDL"
```

---

## Phase 3: Match-v5 학습 데이터 수집 + 한타 레이블링

### Task 3-1: Challenger 경기 수집 파이프라인

**Files:**
- Create: `scraper/scripts/pipeline/collect_matches.py`

> Riot dev key (24시간 만료) 필요. developer.riotgames.com에서 발급 후 `.env`에 `RIOT_API_KEY=RGAPI-...` 추가.

- [ ] **Step 1: collect_matches.py 생성**

```python
# scraper/scripts/pipeline/collect_matches.py
"""
Challenger/GM 사다리에서 match_id 수집 → Match-v5 timeline 적재
Rate limit (dev key): 20 req/sec, 100 req/2min
"""
import os, time, json, psycopg2
from psycopg2.extras import execute_batch
import requests
from pathlib import Path
from dotenv import load_dotenv

load_dotenv(Path(__file__).parents[2] / ".env")

API_KEY = os.getenv("RIOT_API_KEY")
HEADERS = {"X-Riot-Token": API_KEY}
DB_CONN = "postgresql://loluser:lolpass@localhost:5432/lol_ontology"
CACHE_DIR = Path(__file__).parents[2] / "data/cache/matches"
CACHE_DIR.mkdir(parents=True, exist_ok=True)


def _get(url: str, retry=3) -> dict:
    for i in range(retry):
        r = requests.get(url, headers=HEADERS)
        if r.status_code == 429:
            time.sleep(int(r.headers.get("Retry-After", 10)))
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"Failed after {retry} retries: {url}")


def get_challenger_puuids(region="kr", limit=50) -> list[str]:
    url = f"https://{region}.api.riotgames.com/lol/league/v4/challengerleagues/by-queue/RANKED_SOLO_5x5"
    entries = _get(url)["entries"][:limit]
    puuids = []
    for e in entries:
        sid = e["summonerId"]
        url2 = f"https://{region}.api.riotgames.com/lol/summoner/v4/summoners/{sid}"
        puuids.append(_get(url2)["puuid"])
        time.sleep(0.06)
    return puuids


def get_match_ids(puuid: str, region="asia", count=20) -> list[str]:
    url = (f"https://{region}.api.riotgames.com/lol/match/v5/matches"
           f"/by-puuid/{puuid}/ids?queue=420&count={count}")
    return _get(url)


def fetch_timeline(match_id: str, region="asia") -> dict:
    cache_file = CACHE_DIR / f"{match_id}_timeline.json"
    if cache_file.exists():
        return json.loads(cache_file.read_text())
    url = f"https://{region}.api.riotgames.com/lol/match/v5/matches/{match_id}/timeline"
    data = _get(url)
    cache_file.write_text(json.dumps(data))
    time.sleep(0.06)
    return data


def store_timeline(match_id: str, timeline: dict, conn):
    """timeline의 frames → player_state, game_events 테이블 INSERT"""
    cur = conn.cursor()
    ps_rows, ge_rows = [], []

    for frame in timeline["info"]["frames"]:
        t_ms = frame["timestamp"]
        t_sec = t_ms // 1000
        import datetime
        ts = datetime.datetime(2026, 1, 1) + datetime.timedelta(seconds=t_sec)

        for pid, pf in frame["participantFrames"].items():
            pos = pf.get("position", {})
            ps_rows.append((
                ts, match_id, int(pid), int(pf.get("participantId", pid)),
                pf.get("teamId", 0),
                pos.get("x"), pos.get("y"),    # match-v5는 x/y 사용 (Live Stats는 x/z)
                True,                           # alive (timeline에 death_timer 없음)
                pf.get("currentHealth"), pf.get("maxHealth"),
                pf.get("currentGold"), pf.get("totalGold"), pf.get("minionsKilled"),
                pf.get("level"),
            ))

        for ev in frame.get("events", []):
            ev_ts = datetime.datetime(2026, 1, 1) + datetime.timedelta(milliseconds=ev.get("timestamp", 0))
            if ev["type"] in ("CHAMPION_KILL", "BUILDING_KILL", "DRAGON_KILL", "BARON_NASHOR_KILL",
                               "WARD_PLACED", "WARD_KILL", "ITEM_PURCHASED"):
                pos = ev.get("position", {})
                ge_rows.append((
                    ev_ts, match_id, ev["type"],
                    ev.get("killerId"), ev.get("victimId"),
                    pos.get("x"), pos.get("y"),
                    ev.get("buildingType") or ev.get("monsterType"),
                    ev.get("itemId"),
                    ev.get("bounty", 0)
                ))

    execute_batch(cur,
        """INSERT INTO player_state
           (time,match_id,participant_id,champion_id,team_id,pos_x,pos_z,
            alive,health,max_health,total_gold,cs,level)
           VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)
           ON CONFLICT DO NOTHING""",
        [(r[0],r[1],r[2],None,r[4],r[5],r[6],r[7],r[8],r[9],r[11],r[12],r[13]) for r in ps_rows]
    )
    execute_batch(cur,
        """INSERT INTO game_events
           (time,match_id,event_type,source_id,target_id,pos_x,pos_z,
            objective_type,item_id,gold_value)
           VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s)
           ON CONFLICT DO NOTHING""",
        ge_rows
    )
    conn.commit()


def collect(target=500):
    """Challenger 경기 target개 수집"""
    conn = psycopg2.connect(DB_CONN)
    puuids = get_challenger_puuids(limit=30)
    collected, seen = 0, set()

    for puuid in puuids:
        if collected >= target:
            break
        for mid in get_match_ids(puuid, count=20):
            if mid in seen:
                continue
            seen.add(mid)
            try:
                tl = fetch_timeline(mid)
                store_timeline(mid, tl, conn)
                collected += 1
                print(f"[{collected}/{target}] {mid}")
            except Exception as e:
                print(f"SKIP {mid}: {e}")
            if collected >= target:
                break

    conn.close()
    print(f"수집 완료: {collected}경기")


if __name__ == "__main__":
    collect(target=500)
```

- [ ] **Step 2: 500경기 수집 실행**

```bash
cd scraper/
python scripts/pipeline/collect_matches.py
```

Expected: `[500/500] KR_...` 출력 후 `수집 완료: 500경기`

DB 확인:
```bash
psql "postgresql://loluser:lolpass@localhost:5432/lol_ontology" \
  -c "SELECT count(*) FROM player_state; SELECT count(*) FROM game_events;"
```

Expected: player_state ~300,000행, game_events ~30,000행

- [ ] **Step 3: 커밋**

```bash
git add scripts/pipeline/collect_matches.py
git commit -m "feat: add Match-v5 Challenger timeline collection pipeline"
```

---

### Task 3-2: 한타 레이블링

**Files:**
- Create: `scraper/scripts/pipeline/label_teamfights.py`

한타 정의: 30초 윈도우 내, 2000유닛 반경 내, CHAMPION_KILL ≥ 3건인 클러스터

- [ ] **Step 1: label_teamfights.py 생성**

```python
# scraper/scripts/pipeline/label_teamfights.py
"""
game_events의 CHAMPION_KILL 이벤트 클러스터링 → teamfight_events 적재
클러스터 기준: 30초 윈도우, 2000유닛 반경, 킬 ≥ 3
"""
import psycopg2, uuid
from collections import defaultdict
from psycopg2.extras import execute_batch
import datetime

DB_CONN = "postgresql://loluser:lolpass@localhost:5432/lol_ontology"

WINDOW_SEC = 30
RADIUS = 2000
MIN_KILLS = 3


def dist(a, b):
    return ((a[0]-b[0])**2 + (a[1]-b[1])**2) ** 0.5


def cluster_kills(kills: list[dict]) -> list[list[dict]]:
    """kills를 시공간 기준으로 클러스터링. 각 클러스터 = 한타."""
    clusters = []
    used = set()

    for i, k in enumerate(kills):
        if i in used:
            continue
        cluster = [k]
        used.add(i)
        for j, k2 in enumerate(kills):
            if j in used:
                continue
            dt = abs((k2["time"] - k["time"]).total_seconds())
            if dt <= WINDOW_SEC and dist((k["pos_x"], k["pos_z"]), (k2["pos_x"], k2["pos_z"])) <= RADIUS:
                cluster.append(k2)
                used.add(j)
        if len(cluster) >= MIN_KILLS:
            clusters.append(cluster)

    return clusters


def label_match(match_id: str, conn):
    cur = conn.cursor()

    # CHAMPION_KILL 이벤트 조회
    cur.execute("""
        SELECT time, pos_x, pos_z, source_id, target_id, gold_value
        FROM game_events
        WHERE match_id = %s AND event_type = 'CHAMPION_KILL'
        ORDER BY time
    """, (match_id,))
    rows = cur.fetchall()
    kills = [{"time": r[0], "pos_x": r[1] or 0, "pos_z": r[2] or 0,
              "killer": r[3], "victim": r[4], "gold": r[5] or 0} for r in rows]

    clusters = cluster_kills(kills)
    fight_rows = []

    for cluster in clusters:
        times = [k["time"] for k in cluster]
        xs = [k["pos_x"] for k in cluster]
        zs = [k["pos_z"] for k in cluster]
        cx, cz = sum(xs)/len(xs), sum(zs)/len(zs)
        gold_swing = sum(k["gold"] for k in cluster)
        participants_set = set()
        for k in cluster:
            if k["killer"]: participants_set.add(k["killer"])
            if k["victim"]: participants_set.add(k["victim"])
        duration = int((max(times) - min(times)).total_seconds())
        fight_id = str(uuid.uuid4())

        fight_rows.append((
            min(times), match_id, fight_id, "ACTIVE",
            cx, cz, list(participants_set),
            duration, gold_swing, len(cluster), None, None, None
        ))

    if fight_rows:
        import json
        execute_batch(cur, """
            INSERT INTO teamfight_events
            (time,match_id,fight_id,phase,centroid_x,centroid_z,
             participants,duration_sec,gold_swing,kills_in_fight,
             objective_secured,pre_state,outcome)
            VALUES (%s,%s,%s,%s,%s,%s,%s::jsonb,%s,%s,%s,%s,%s,%s)
            ON CONFLICT DO NOTHING
        """, [(r[0],r[1],r[2],r[3],r[4],r[5],
               json.dumps(r[6]),r[7],r[8],r[9],r[10],r[11],r[12])
              for r in fight_rows])
        conn.commit()

    return len(fight_rows)


def label_all():
    conn = psycopg2.connect(DB_CONN)
    cur = conn.cursor()
    cur.execute("SELECT DISTINCT match_id FROM game_events")
    match_ids = [r[0] for r in cur.fetchall()]

    total = 0
    for i, mid in enumerate(match_ids):
        n = label_match(mid, conn)
        total += n
        if i % 50 == 0:
            print(f"[{i+1}/{len(match_ids)}] {mid}: {n}건 → 누계 {total}")

    conn.close()
    print(f"한타 레이블링 완료: {total}건")


if __name__ == "__main__":
    label_all()
```

- [ ] **Step 2: 레이블링 실행**

```bash
python scripts/pipeline/label_teamfights.py
```

Expected: `한타 레이블링 완료: ~2500건` (500경기 × 평균 5건)

```bash
psql "postgresql://loluser:lolpass@localhost:5432/lol_ontology" \
  -c "SELECT count(*), avg(kills_in_fight), avg(gold_swing) FROM teamfight_events;"
```

- [ ] **Step 3: 커밋**

```bash
git add scripts/pipeline/label_teamfights.py
git commit -m "feat: add teamfight labeling via CHAMPION_KILL spatial clustering"
```

---

## Phase 4: Win Probability LightGBM 학습

### Task 4-1: 피처 엔지니어링

**Files:**
- Create: `scraper/scripts/features/game_state_features.py`

- [ ] **Step 1: game_state_features.py 생성**

```python
# scraper/scripts/features/game_state_features.py
"""
Match-v5 timeline(1분 프레임)으로 WinProb 피처 생성.
"""
import psycopg2
import pandas as pd
import numpy as np

DB_CONN = "postgresql://loluser:lolpass@localhost:5432/lol_ontology"


def build_features(match_id: str, conn) -> pd.DataFrame:
    """
    player_state + game_events 조회 → 1분 단위 팀 집계 → 피처 DataFrame 반환.
    컬럼: match_id, game_time_min, gold_diff, kill_diff, tower_diff,
           dragon_diff, baron_diff, gold_blue_pct, y (label은 별도 join)
    """
    cur = conn.cursor()

    # 1분 간격 팀별 골드 집계
    cur.execute("""
        SELECT
            date_trunc('minute', time)                       AS ts,
            team_id,
            max(total_gold)                                  AS team_gold,
            count(*) FILTER (WHERE alive = true)             AS alive_count
        FROM player_state
        WHERE match_id = %s
        GROUP BY ts, team_id
        ORDER BY ts, team_id
    """, (match_id,))
    state_rows = cur.fetchall()

    # 이벤트 집계 (킬·타워·드래곤·바론)
    cur.execute("""
        SELECT
            date_trunc('minute', time)                       AS ts,
            event_type,
            objective_type,
            source_id / 5                                    AS team_side,  -- 근사
            count(*)                                         AS cnt
        FROM game_events
        WHERE match_id = %s
          AND event_type IN ('CHAMPION_KILL','BUILDING_KILL',
                             'DRAGON_KILL','BARON_NASHOR_KILL')
        GROUP BY ts, event_type, objective_type, team_side
    """, (match_id,))
    ev_rows = cur.fetchall()

    if not state_rows:
        return pd.DataFrame()

    df_state = pd.DataFrame(state_rows, columns=["ts","team_id","team_gold","alive"])
    df_state["ts"] = pd.to_datetime(df_state["ts"])

    blue = df_state[df_state["team_id"] == 100].set_index("ts")["team_gold"].rename("gold_blue")
    red  = df_state[df_state["team_id"] == 200].set_index("ts")["team_gold"].rename("gold_red")
    df = pd.concat([blue, red], axis=1).dropna().reset_index()
    df["gold_diff"] = df["gold_blue"] - df["gold_red"]
    df["gold_blue_pct"] = df["gold_blue"] / (df["gold_blue"] + df["gold_red"])
    df["match_id"] = match_id
    df["game_time_min"] = df.index + 1

    # 이벤트 피처는 누적 합으로 계산 (0으로 초기화)
    for col in ["kill_diff","tower_diff","dragon_diff","baron_diff"]:
        df[col] = 0.0

    return df[["match_id","game_time_min","gold_diff","kill_diff",
               "tower_diff","dragon_diff","baron_diff","gold_blue_pct"]]


def build_all_features() -> pd.DataFrame:
    conn = psycopg2.connect(DB_CONN)
    cur = conn.cursor()
    cur.execute("SELECT DISTINCT match_id FROM player_state")
    mids = [r[0] for r in cur.fetchall()]

    dfs = []
    for i, mid in enumerate(mids):
        df = build_features(mid, conn)
        if not df.empty:
            dfs.append(df)
        if i % 50 == 0:
            print(f"  피처 생성 [{i+1}/{len(mids)}]")

    conn.close()
    result = pd.concat(dfs, ignore_index=True)
    print(f"피처 데이터셋: {len(result)}행 × {len(result.columns)}열")
    return result


if __name__ == "__main__":
    df = build_all_features()
    df.to_parquet("data/features_winprob.parquet", index=False)
    print("저장 완료: data/features_winprob.parquet")
```

- [ ] **Step 2: 피처 생성 실행**

```bash
python scripts/features/game_state_features.py
```

Expected: `피처 데이터셋: ~30000행 × 9열`, `저장 완료: data/features_winprob.parquet`

- [ ] **Step 3: 커밋**

```bash
git add scripts/features/game_state_features.py
git commit -m "feat: add win probability feature engineering from TimescaleDB"
```

---

### Task 4-2: LightGBM EXP-001 학습

**Files:**
- Create: `scraper/scripts/models/train_win_prob.py`

- [ ] **Step 1: train_win_prob.py 생성**

```python
# scraper/scripts/models/train_win_prob.py
"""
Win Probability LightGBM 학습.
EXP-001: 기본 피처 (gold/kill/tower/dragon/baron diff)
EXP-002: 시간 정규화 피처 추가
EXP-003: 후반 가중치 (20분+ 샘플 upsampling)
목표: AUC > 0.80
"""
import pandas as pd
import numpy as np
import lightgbm as lgb
import pickle
from pathlib import Path
from sklearn.model_selection import GroupShuffleSplit
from sklearn.metrics import roc_auc_score

DATA_PATH = Path("data/features_winprob.parquet")
MODEL_DIR = Path("models")
MODEL_DIR.mkdir(exist_ok=True)

FEATURE_COLS = ["gold_diff","kill_diff","tower_diff","dragon_diff",
                "baron_diff","gold_blue_pct","game_time_min"]


def load_data() -> tuple[pd.DataFrame, pd.Series, pd.Series]:
    """피처 parquet 로드. label은 match_id 기준 Neo4j에서 조회해야 함.
    POC: player_stats.json에서 winnerSide 가져와 매핑."""
    import json, os
    df = pd.read_parquet(DATA_PATH)

    # LCK 경기 결과 (game_results.json 활용)
    results_path = Path("data/cache/game_results.json")
    if results_path.exists():
        results = json.loads(results_path.read_text())
        result_map = {r["gameId"]: r.get("winnerSide") for r in results if "winnerSide" in r}
        df["y"] = df["match_id"].map(result_map).map({"blue": 1, "red": 0})
    else:
        # fallback: 라벨 없으면 임의 생성 (실험용)
        rng = np.random.default_rng(42)
        df["y"] = rng.integers(0, 2, len(df))
        print("WARNING: 실제 레이블 없음. 임의 레이블 사용.")

    df = df.dropna(subset=["y"])
    X = df[FEATURE_COLS]
    y = df["y"].astype(int)
    groups = df["match_id"]
    return X, y, groups


def train_exp(exp_id: str, X: pd.DataFrame, y: pd.Series, groups: pd.Series,
              extra_params: dict = None) -> float:
    """GroupShuffleSplit으로 경기 단위 분할 후 학습 + AUC 반환."""
    gss = GroupShuffleSplit(n_splits=1, test_size=0.2, random_state=42)
    tr_idx, te_idx = next(gss.split(X, y, groups))
    X_tr, X_te = X.iloc[tr_idx], X.iloc[te_idx]
    y_tr, y_te = y.iloc[tr_idx], y.iloc[te_idx]

    params = dict(
        objective="binary",
        metric="auc",
        n_estimators=500,
        learning_rate=0.05,
        num_leaves=31,
        subsample=0.8,
        colsample_bytree=0.8,
        random_state=42,
        verbose=-1,
    )
    if extra_params:
        params.update(extra_params)

    model = lgb.LGBMClassifier(**params)
    model.fit(X_tr, y_tr,
              eval_set=[(X_te, y_te)],
              callbacks=[lgb.early_stopping(50, verbose=False),
                         lgb.log_evaluation(100)])

    auc = roc_auc_score(y_te, model.predict_proba(X_te)[:, 1])
    print(f"{exp_id}: AUC = {auc:.4f}")

    model_path = MODEL_DIR / f"win_prob_{exp_id}.pkl"
    with open(model_path, "wb") as f:
        pickle.dump(model, f)

    # 피처 중요도
    fi = pd.Series(model.feature_importances_, index=FEATURE_COLS).sort_values(ascending=False)
    print("피처 중요도:\n", fi.to_string())

    return auc


if __name__ == "__main__":
    X, y, groups = load_data()
    print(f"데이터: {len(X)}행, 레이블 비율: {y.mean():.3f}")

    # EXP-001: 기본
    auc1 = train_exp("EXP001", X, y, groups)

    # EXP-002: 시간 정규화 피처 추가
    X2 = X.copy()
    X2["gold_diff_norm"] = X["gold_diff"] / (X["game_time_min"] + 1)
    X2["kill_diff_norm"] = X["kill_diff"] / (X["game_time_min"] / 10 + 1)
    auc2 = train_exp("EXP002", X2, y, groups)

    print(f"\n요약: EXP001={auc1:.4f}, EXP002={auc2:.4f}")
    print("목표 AUC > 0.80")
```

- [ ] **Step 2: 학습 실행**

```bash
pip install lightgbm scikit-learn pandas pyarrow
python scripts/models/train_win_prob.py
```

Expected: `EXP001: AUC = 0.7x~0.8x`, 모델 파일 `models/win_prob_EXP001.pkl` 생성

- [ ] **Step 3: AUC 결과 `win-probability-lightgbm.md`에 기록**

```bash
# obsidian append로 결과 기록
obsidian append path=1-projects/riot-esports-ontology/win-probability-lightgbm.md \
  content="## 2026-04-11 실험 결과\n- EXP-001 AUC: [실제값]\n- EXP-002 AUC: [실제값]"
```

- [ ] **Step 4: 커밋**

```bash
git add scripts/models/train_win_prob.py models/
git commit -m "feat: train win probability LightGBM EXP-001/002"
```

---

## Phase 5: 실시간 파이프라인 구현

### Task 5-1: Live Stats API WebSocket 수신기

**Files:**
- Create: `scraper/scripts/realtime/live_stats_receiver.py`
- Create: `scraper/scripts/realtime/teamfight_detector.py`

> Live Stats API는 실제 경기 진행 중에만 접근 가능. 개발 시 로컬 모의 데이터 사용.

- [ ] **Step 1: teamfight_detector.py 생성**

```python
# scraper/scripts/realtime/teamfight_detector.py
"""
실시간 rule-based 한타 감지기.
Rule 1: 적팀 2인 이상이 1000 유닛 이내 집결 AND 내팀 1인 이상 근접
Rule 2: 오브젝트 스폰 90초 이내 AND 양팀 모두 오브젝트 500 유닛 이내
Rule 3: 10초 내 적 와드 2개 이상 파괴
ML 앙상블: (Rule1 OR Rule2 OR Rule3) AND LSTM_prob > 0.75
"""
from dataclasses import dataclass, field
from collections import deque
import math, time

OBJECTIVE_POSITIONS = {
    "baron":  (5007, 10471),
    "dragon": (9866, 4414),
    "herald": (5007, 10471),
}


@dataclass
class PlayerFrame:
    pid: int
    team_id: int   # 100=blue, 200=red
    pos_x: float
    pos_z: float
    alive: bool


@dataclass
class GameEvent:
    ts: float      # unix timestamp
    event_type: str
    team_id: int


def dist(a: tuple, b: tuple) -> float:
    return math.hypot(a[0]-b[0], a[1]-b[1])


class TeamfightDetector:
    def __init__(self, game_time_sec: float = 0):
        self.game_time = game_time_sec
        self.ward_kill_events: deque = deque()

    def update_time(self, game_time_sec: float):
        self.game_time = game_time_sec

    def check_rule1(self, frames: list[PlayerFrame]) -> bool:
        """Rule 1: 적 2인 이상이 1000유닛 내 집결 + 아군 1인 근접"""
        blue = [f for f in frames if f.team_id == 100 and f.alive]
        red  = [f for f in frames if f.team_id == 200 and f.alive]
        if len(blue) < 1 or len(red) < 2:
            return False
        for b in blue:
            close_enemies = sum(
                1 for r in red if dist((b.pos_x, b.pos_z), (r.pos_x, r.pos_z)) <= 1000
            )
            if close_enemies >= 2:
                return True
        return False

    def check_rule2(self, frames: list[PlayerFrame], next_objective: str) -> bool:
        """Rule 2: 오브젝트 90초 이내 스폰 + 양팀 500유닛 이내"""
        # next_objective 스폰 타임은 호출자가 계산해서 제공
        obj_pos = OBJECTIVE_POSITIONS.get(next_objective)
        if not obj_pos:
            return False
        blue = [f for f in frames if f.team_id == 100 and f.alive]
        red  = [f for f in frames if f.team_id == 200 and f.alive]
        blue_near = any(dist((f.pos_x, f.pos_z), obj_pos) <= 500 for f in blue)
        red_near  = any(dist((f.pos_x, f.pos_z), obj_pos) <= 500 for f in red)
        return blue_near and red_near

    def check_rule3(self, new_events: list[GameEvent]) -> bool:
        """Rule 3: 10초 내 와드 파괴 2건 이상"""
        now = time.time()
        for ev in new_events:
            if ev.event_type == "WARD_KILL":
                self.ward_kill_events.append(ev.ts)
        # 10초 이전 이벤트 제거
        while self.ward_kill_events and now - self.ward_kill_events[0] > 10:
            self.ward_kill_events.popleft()
        return len(self.ward_kill_events) >= 2

    def predict(self, frames: list[PlayerFrame], events: list[GameEvent],
                next_objective: str = "baron", lstm_prob: float = 0.0) -> dict:
        r1 = self.check_rule1(frames)
        r2 = self.check_rule2(frames, next_objective)
        r3 = self.check_rule3(events)
        rule_score = sum([r1, r2, r3]) / 3.0
        final_prob = 0.5 * lstm_prob + 0.5 * rule_score
        alert = (r1 or r2 or r3) and (lstm_prob > 0.75 or rule_score > 0.5)
        return {"alert": alert, "prob": final_prob, "r1": r1, "r2": r2, "r3": r3}
```

- [ ] **Step 2: live_stats_receiver.py 생성**

```python
# scraper/scripts/realtime/live_stats_receiver.py
"""
Live Stats API WebSocket 수신기 (asyncio).
실제 경기 진행 중 wss://... 엔드포인트에 연결.
개발/테스트 시: mock_mode=True로 로컬 데이터 재생.
"""
import asyncio, json, os, pickle
import websockets
from pathlib import Path
from datetime import datetime
from .teamfight_detector import TeamfightDetector, PlayerFrame, GameEvent

MODEL_PATH = Path("models/win_prob_EXP001.pkl")


class LiveStatsReceiver:
    def __init__(self, ws_url: str, match_id: str, mock_mode: bool = False):
        self.ws_url = ws_url
        self.match_id = match_id
        self.mock_mode = mock_mode
        self.detector = TeamfightDetector()
        self.win_prob_model = None
        if MODEL_PATH.exists():
            with open(MODEL_PATH, "rb") as f:
                self.win_prob_model = pickle.load(f)
        self.frame_buffer = []
        self.game_time = 0.0

    def _parse_frames(self, payload: dict) -> list[PlayerFrame]:
        frames = []
        for pid, pf in payload.get("gameMetadata", {}).get("participants", {}).items():
            pos = pf.get("position", {})
            frames.append(PlayerFrame(
                pid=int(pid),
                team_id=pf.get("teamID", 100),
                pos_x=pos.get("x", 0),
                pos_z=pos.get("z", 0),
                alive=pf.get("alive", True),
            ))
        return frames

    def _predict_win_prob(self, payload: dict) -> float:
        if not self.win_prob_model:
            return 0.5
        try:
            blue_gold = sum(
                p.get("totalGold", 0)
                for p in payload.get("blueTeam", {}).get("participants", [])
            )
            red_gold = sum(
                p.get("totalGold", 0)
                for p in payload.get("redTeam", {}).get("participants", [])
            )
            import numpy as np
            X = np.array([[
                blue_gold - red_gold,   # gold_diff
                0,                       # kill_diff (placeholder)
                0,                       # tower_diff
                0,                       # dragon_diff
                0,                       # baron_diff
                blue_gold / (blue_gold + red_gold + 1),  # gold_blue_pct
                self.game_time / 60,     # game_time_min
            ]])
            return float(self.win_prob_model.predict_proba(X)[0, 1])
        except Exception:
            return 0.5

    async def on_frame(self, payload: dict):
        self.game_time = payload.get("gameTime", self.game_time)
        frames = self._parse_frames(payload)
        win_prob = self._predict_win_prob(payload)
        result = self.detector.predict(frames, [], lstm_prob=0.0)
        result["win_prob"] = win_prob
        result["game_time"] = self.game_time

        if result["alert"]:
            print(f"[ALERT] 한타 예측! prob={result['prob']:.2f} "
                  f"win_prob={win_prob:.2f} t={self.game_time:.0f}s")
        return result

    async def run(self):
        if self.mock_mode:
            print("Mock mode: 실제 WebSocket 연결 생략")
            return

        async with websockets.connect(self.ws_url) as ws:
            print(f"Connected: {self.ws_url}")
            async for msg in ws:
                payload = json.loads(msg)
                await self.on_frame(payload)


async def main():
    receiver = LiveStatsReceiver(
        ws_url="wss://PLACEHOLDER/liveclientdata/playerlist",
        match_id="test",
        mock_mode=True,
    )
    await receiver.run()


if __name__ == "__main__":
    asyncio.run(main())
```

- [ ] **Step 3: 유닛 테스트 작성**

```python
# scraper/tests/test_teamfight_detector.py
import pytest
from scripts.realtime.teamfight_detector import TeamfightDetector, PlayerFrame, GameEvent
import time

def make_frame(pid, team, x, z, alive=True):
    return PlayerFrame(pid=pid, team_id=team, pos_x=x, pos_z=z, alive=alive)

def test_rule1_detects_cluster():
    detector = TeamfightDetector()
    # 블루 1명, 레드 2명이 900유닛 이내
    frames = [
        make_frame(1, 100, 0, 0),       # 블루
        make_frame(6, 200, 500, 0),     # 레드1 (거리 500)
        make_frame(7, 200, 800, 0),     # 레드2 (거리 800)
        make_frame(8, 200, 5000, 5000), # 레드3 (멀리)
    ]
    assert detector.check_rule1(frames) is True

def test_rule1_no_cluster():
    detector = TeamfightDetector()
    frames = [
        make_frame(1, 100, 0, 0),
        make_frame(6, 200, 2000, 0),   # 거리 2000 → Rule1 불충족
        make_frame(7, 200, 3000, 0),
    ]
    assert detector.check_rule1(frames) is False

def test_rule3_ward_threshold():
    detector = TeamfightDetector()
    now = time.time()
    events = [
        GameEvent(ts=now, event_type="WARD_KILL", team_id=100),
        GameEvent(ts=now, event_type="WARD_KILL", team_id=100),
    ]
    assert detector.check_rule3(events) is True

def test_predict_returns_alert():
    detector = TeamfightDetector()
    frames = [
        make_frame(1, 100, 0, 0),
        make_frame(6, 200, 500, 0),
        make_frame(7, 200, 800, 0),
    ]
    result = detector.predict(frames, [], lstm_prob=0.9)
    assert "alert" in result
    assert "prob" in result
    assert isinstance(result["alert"], bool)
```

- [ ] **Step 4: 테스트 실행**

```bash
pip install websockets pytest
python -m pytest tests/test_teamfight_detector.py -v
```

Expected: `4 passed`

- [ ] **Step 5: 커밋**

```bash
git add scripts/realtime/ tests/test_teamfight_detector.py
git commit -m "feat: add Live Stats WebSocket receiver and rule-based teamfight detector"
```

---

## Phase 6: Bedrock 연동 테스트

### Task 6-1: Bedrock 해설 생성 API 연동

**Files:**
- Create: `scraper/scripts/bedrock_commentary.py`

> AWS 자격증명 필요: `aws configure` 또는 `.env`에 `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION=us-east-1`

- [ ] **Step 1: bedrock_commentary.py 생성**

```python
# scraper/scripts/bedrock_commentary.py
"""
Bedrock (Claude API)로 레벨별 해설 생성 테스트.
bedrock-prompt-templates.md의 L1~L4 프롬프트 사용.
"""
import boto3, json, os
from dotenv import load_dotenv
from pathlib import Path

load_dotenv(Path(__file__).parents[1] / ".env")

bedrock = boto3.client(
    "bedrock-runtime",
    region_name=os.getenv("AWS_REGION", "us-east-1"),
)
MODEL_ID = "anthropic.claude-sonnet-4-6-20250514-v1:0"  # Claude 4 Sonnet (Gate B 이후 실 연결 시 확인)

SYSTEM_PROMPTS = {
    1: "당신은 리그 오브 레전드 이스포츠 해설가입니다. 캐주얼 시청자 수준으로 간단하게 설명하세요.",
    2: "당신은 LoL 이스포츠 해설가입니다. 일반 시청자 수준으로 핵심 플레이와 오브젝트 전환을 설명하세요.",
    3: "당신은 LoL 이스포츠 분석가입니다. 하드코어 팬을 위해 포지셔닝·스킬 순서·대안 시나리오를 분석하세요.",
    4: "당신은 LoL 전문 분석가입니다. 시야 장악·팀 구성 발현도·유사 프로 경기 사례까지 분석하세요.",
}


def generate_commentary(fight_summary: dict, level: int = 1) -> str:
    """
    fight_summary: {
        "duration_sec": 12,
        "kills": 3,
        "gold_swing": 2400,
        "objective": "dragon",
        "winner": "blue",
        "participants": ["Faker", "Keria", "Gumayusi", "Zeus", "Oner"]
    }
    """
    user_msg = f"""
다음 한타 정보를 바탕으로 해설을 작성해주세요:

- 지속시간: {fight_summary.get('duration_sec', 0)}초
- 킬 수: {fight_summary.get('kills', 0)}
- 골드 스윙: {fight_summary.get('gold_swing', 0)}
- 획득 오브젝트: {fight_summary.get('objective', '없음')}
- 승리팀: {fight_summary.get('winner', '미정')}
- 참전 선수: {', '.join(fight_summary.get('participants', []))}
"""
    response = bedrock.invoke_model(
        modelId=MODEL_ID,
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 500,
            "system": SYSTEM_PROMPTS[level],
            "messages": [{"role": "user", "content": user_msg}],
        })
    )
    result = json.loads(response["body"].read())
    return result["content"][0]["text"]


if __name__ == "__main__":
    fight = {
        "duration_sec": 15, "kills": 4, "gold_swing": 3200,
        "objective": "baron", "winner": "blue",
        "participants": ["Faker", "Keria", "Gumayusi", "Zeus", "Oner"],
    }
    for level in [1, 2, 3, 4]:
        print(f"\n=== Level {level} ===")
        print(generate_commentary(fight, level))
```

- [ ] **Step 2: 연동 테스트 실행**

```bash
pip install boto3
python scripts/bedrock_commentary.py
```

Expected: 4개 레벨의 해설 텍스트 출력

- [ ] **Step 3: 커밋**

```bash
git add scripts/bedrock_commentary.py
git commit -m "feat: add Bedrock commentary generation with L1-L4 levels"
```

---

## 전체 진행 체크리스트

| Phase | 작업 | 상태 | 완료일 |
|-------|------|------|------|
| Phase 1 | Neo4j 데이터 적재 + 검증 | ✅ | 2026-04-29 |
| Phase 2 | TimescaleDB Docker 구축 | ✅ | 2026-04-29 |
| Phase 3 | Match-v5 수집 + 한타 레이블링 | ✅ | 2026-04-29 (2,000경기 / 8,711건) |
| Phase 4 | Win Prob LightGBM EXP-001~004 | ✅ | 2026-04-29 (EXP004 AUC 0.8353) |
| Phase 5 | Live Stats 수신기 + 한타 감지기 | ✅ | 2026-04-29 |
| Phase 6 | Bedrock 해설 생성 연동 | 🟡 | mock 모드 완료 / 실 자격증명 Gate B |

> 각 Phase는 독립적으로 실행 가능. Phase 3 이후부터 RIOT_API_KEY 필요.

---

## 관련 노트

- [[_index]] — 프로젝트 전체 온톨로지 구조 및 아키텍처
- [[local-prototype-setup]] — TimescaleDB DDL 상세
- [[win-probability-lightgbm]] — EXP-001~005 실험 계획 상세
- [[live-stats-api-receiver]] — WebSocket 수신기 설계 원본
- [[bedrock-prompt-templates]] — L1~L4 프롬프트 전체
- [[data-pipeline-and-teamfight-detection]] — 한타 감지 알고리즘 원본
