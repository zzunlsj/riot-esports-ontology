---
type: project
date: 2026-03-23
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [ontology, AWS, esports, LoL, knowledge-graph, real-time, teamfight]
---

# Riot Esports 실시간 경기 맥락 온톨로지

## 프로젝트 목표

- **1차 목표**: 한타 발생 전 관전 시점·관전 포인트 자동 포착
- **2차 목표**: 한타 종료 후 유저 수준별 딥다이브 해설 생성
- **3차 목표**: 사전·인게임 승부 예측 (Pre-game / Win Probability / Teamfight Outcome)
- **기술 스택**: Riot Esports API + AWS (Neptune · Kinesis · SageMaker · Bedrock) + TimescaleDB + pgvectorscale
- **현재 프로토타입 DB**: Neo4j Desktop (로컬) — Neptune 이전 전 로컬 검증 단계

## 데이터 거버넌스

- Riot API / lolesports / Data Dragon만 canonical raw로 취급한다.
- 모델 학습셋은 공식 raw + raw 기반 verified derived만 사용한다.
- 비공식 보조 소스는 현재 운영 경로에서 제외한다.

---

## 온톨로지 구조 개요

### 레이어 구분

```
┌──────────────────────────────────────────────────────┐
│  LAYER 3 : 추론·생성 레이어                            │  SageMaker + Bedrock
│  한타 예측 / 승부 예측 / 해설 생성 엔진                 │
├──────────────────────────────────────────────────────┤
│  LAYER 2B : 벡터 레이어                               │  pgvectorscale (PostgreSQL ext.)
│  게임 상태 임베딩 · 구성 벡터 · 유사 상황 검색          │
├──────────────────────────────────────────────────────┤
│  LAYER 2A : 동적 온톨로지                              │  TimescaleDB (PostgreSQL ext.)
│  실시간 게임 상태 (시계열) — 동일 DB 인스턴스           │
├──────────────────────────────────────────────────────┤
│  LAYER 1 : 정적 온톨로지                               │  Neptune (Property Graph)
│  챔피언·아이템·팀·선수 지식 그래프                      │
└──────────────────────────────────────────────────────┘

핵심: Layer 2A + 2B가 같은 PostgreSQL 인스턴스 위에서 동작
→ 시계열 쿼리 + 벡터 유사도 검색을 단일 SQL로 수행 가능
→ 별도 벡터 DB(Pinecone 등) 불필요
```

---

## LAYER 1 : 정적 온톨로지

Amazon Neptune (Property Graph, openCypher)으로 관리. 패치마다 업데이트되지만 경기 중에는 불변.

### 클래스 정의

#### Champion
```
Champion {
  id: string           // "Jinx", "Leona"
  role: [top|jg|mid|bot|sup]
  class: [assassin|fighter|mage|marksman|support|tank]
  attackRange: number  // 근거리(melee) < 300 / 원거리 >= 300
  baseStats: {
    hp, mana, armor, magicResist, attackDamage, movementSpeed
  }
  abilities: [Ability]
  scalingType: [AD|AP|hybrid|tank]
  engagePotential: low|medium|high   // 온톨로지 추론용
  ccTypes: [stun|slow|knockup|silence|charm|suppression]
}
```

#### Ability
```
Ability {
  id: string
  slot: Q|W|E|R|Passive
  cooldownBase: [number]    // 레벨별 쿨타임 배열
  castRange: number
  targetType: point|direction|unit|aoe|global
  damageType: physical|magic|true|none
  ccType: string|null
  isSkillshot: boolean
  isChanneled: boolean
  interruptible: boolean
  comboRole: initiator|follow-up|peel|execute|sustain
}
```

#### Player
```
Player {
  puuid: string
  gameName: string
  currentRank: {tier, lp}
  mainRoles: [Role]
  championPool: [{championId, gamesPlayed, winRate, kda}]
  recentPerformanceMetrics: {
    avgDamageShare, avgVisionScore, avgKP, avgCSperMin
  }
  playstyle: aggressive|defensive|farm-oriented|roam-oriented
}
```

#### Team
```
Team {
  id: string
  name: string
  region: LCK|LPL|LEC|LCS|...
  roster: [Player]
  teamStyle: {
    earlyGameFocus: score,   // 0-1
    objectivePriority: score,
    teamfightComposition: score,
    splitPushTendency: score
  }
  h2hHistory: [MatchSummary]
}
```

#### MapZone
```
MapZone {
  id: string
  name: top-lane|mid-lane|bot-lane|top-jungle|bot-jungle|
        river-top|river-bot|baron-pit|dragon-pit|base-blue|base-red
  coordinates: {xMin, xMax, yMin, yMax}  // in-game units
  strategicWeight: number   // 시간대별로 달라짐
}
```

#### Objective
```
Objective {
  id: string
  type: dragon|baron|herald|tower|inhibitor|nexus
  spawnTime: number         // 게임 시작 후 초
  respawnInterval: number
  buffEffect: string
  strategicValue: number    // 온톨로지에서 계산
}
```

### 관계(Relationship) 정의

```
(Champion)-[:COUNTERS]->(Champion)           // 카운터 관계
(Champion)-[:SYNERGIZES_WITH]->(Champion)    // 시너지
(Champion)-[:REQUIRES_ITEM]->(Item)          // 핵심 아이템
(Player)-[:MAINS]->(Champion)
(Player)-[:MEMBER_OF]->(Team)
(Ability)-[:CHAINS_WITH]->(Ability)          // CC 콤보 체인
(MapZone)-[:ADJACENT_TO]->(MapZone)
(Objective)-[:LOCATED_IN]->(MapZone)
```

---

## LAYER 2A : 동적 온톨로지

### 시계열 DB 설계 (TimescaleDB)

Amazon Timestream에서 **TimescaleDB**로 변경. 이유: pgvectorscale과 동일 PostgreSQL 인스턴스 공유 → 시계열 + 벡터 검색을 단일 쿼리로 수행. Timescale Cloud(관리형) 또는 RDS for PostgreSQL + 확장으로 운영.

**테이블 1: player_state** — timeline / live state 적재

| Dimension | Type |
|-----------|------|
| match_id | string |
| participant_id | smallint |
| champion_id | smallint |
| team_id | smallint |

| Measure | Type | 설명 |
|---------|------|------|
| pos_x, pos_z | int | 맵 좌표 (`x`,`z` 기준) |
| alive | boolean | 생존 여부 |
| health, max_health | real | 체력 |
| resource, max_resource | real | 마나/자원 |
| level | int | 레벨 |
| total_gold | int | 누적 골드 |
| cs | int | CS |
| q_cd, w_cd, e_cd, r_cd | real | 각 스킬 잔여 쿨타임 |
| flash_cd | real | 플래시 잔여 쿨타임 |
| state_embedding | vector(128) | 상태 임베딩 |

**테이블 2: game_events** — 이벤트 기반

| Measure | Type | 설명 |
|---------|------|------|
| event_type | string | CHAMPION_KILL/BUILDING_KILL/ELITE_MONSTER_KILL/WARD/ITEM |
| source_id | smallint | |
| target_id | smallint | |
| pos_x, pos_z | int | 이벤트 발생 좌표 |
| objective_type | string | 오브젝트 타입 |
| item_id | int | |
| gold_value | int | 킬·오브젝트 골드 환산 |

**테이블 3: team_state** — 팀 단위 상태/예측 시계열

| Measure | Type |
|---------|------|
| total_gold | int |
| kills | int |
| towers | int |
| dragons | int |
| barons | int |
| inhibitors | int |
| cs | int |
| win_prob | real |
| model_signal | real |

**테이블 4: teamfight_events** — 한타 감지 시 생성

| Measure | Type | 설명 |
|---------|------|------|
| fight_id | string | UUID |
| phase | PRE|ACTIVE|POST | 한타 단계 |
| centroid_x, centroid_z | double | 전투 중심 좌표 |
| participants | string | JSON: 참전 선수 목록 |
| duration_sec | int | 한타 지속 시간 |
| gold_swing | int | 골드 변동량 |
| kills_in_fight | int | |
| objective_secured | string | 한타 후 획득 오브젝트 |

현재 로컬 프로토타입에서는:

- Match-v5 timeline 백필로 `player_state`, `game_events`를 학습용으로 채움
- `live_stats_receiver.py` mock replay로 `team_state.win_prob`, `model_signal`을 실시간 추론 시계열처럼 적재
- `win_prob_swings.py`로 경기 간 승률 급변 구간 랭킹을 생성

---

## LAYER 2B : 벡터 레이어 (pgvectorscale)

pgvectorscale = pgvector + StreamingDiskANN 인덱스 + Statistical Binary Quantization
→ pgvector 대비 약 23배 빠른 ANN 검색, 메모리 1/10 수준

### 벡터 임베딩 설계

#### 1. 게임 상태 벡터 (State Snapshot Vector)
```sql
-- player_state 테이블에 vector 컬럼 추가 (TimescaleDB + pgvectorscale 공존)
ALTER TABLE player_state ADD COLUMN state_embedding vector(128);

-- 임베딩 구성 (128차원)
-- [골드차(norm), 킬차(norm), 타워차(norm),
--  팀A 평균HP%, 팀B 평균HP%, 팀A 플래시보유율, 팀B 플래시보유율,
--  팀A R쿨완료율, 팀B R쿨완료율,
--  오브젝트까지거리(팀A), 오브젝트까지거리(팀B),
--  게임경과시간(norm), 챔피언구성벡터(팀A 64dim), 챔피언구성벡터(팀B 64dim)]
```

**활용:** 현재 게임 상태 → 유사 과거 상황 top-K 검색 → 해당 상황에서 한타 발생률·승률 계산

#### 2. 챔피언 구성 벡터 (Composition Embedding)
```sql
CREATE TABLE composition_embeddings (
  comp_id       TEXT PRIMARY KEY,
  team_id       TEXT,
  match_id      TEXT,
  game_time_sec INT,
  comp_vector   vector(64),   -- 5개 챔피언 → 64차원 압축
  win           BOOLEAN,
  fight_occurred BOOLEAN
);

CREATE INDEX ON composition_embeddings
  USING diskann (comp_vector);  -- StreamingDiskANN 인덱스
```

**활용:** "현재 내 구성과 유사한 구성이 상대 구성을 만났을 때 승률" → Pre-game 예측

#### 3. 한타 상태 벡터 (Teamfight Context Vector)
```sql
CREATE TABLE teamfight_embeddings (
  fight_id      TEXT PRIMARY KEY,
  pre_state     vector(128),   -- 한타 직전 게임 상태
  outcome       TEXT,          -- 'team_a_win' | 'team_b_win' | 'draw'
  gold_swing    INT,
  objective     TEXT
);
```

**활용:** 한타 발생 직전 상태 → 유사 과거 한타 검색 → 한타 결과 예측 + 해설 레퍼런스

### 핵심 쿼리 예시

```sql
-- 현재 상태와 유사한 과거 게임 상황 TOP-5 검색 (시계열 + 벡터 단일 쿼리)
SELECT
  s.match_id,
  s.game_time_sec,
  s.state_embedding <=> $current_vector AS distance,
  tf.outcome,
  tf.gold_swing
FROM player_state s
JOIN teamfight_embeddings tf ON s.match_id = tf.fight_id
WHERE s.time > NOW() - INTERVAL '90 days'   -- TimescaleDB 파티션 활용
ORDER BY distance ASC
LIMIT 5;
```

---

## LAYER 3 : 추론·생성 레이어

### 3-1. 한타 예측 모델

**입력 피처셋 (SageMaker)**

| 피처 그룹 | 피처 |
|----------|------|
| 공간 피처 | 팀별 플레이어 평균 좌표, 상대팀 간 거리, 오브젝트 거리 |
| 시간 피처 | 오브젝트 스폰까지 남은 초, 이전 한타로부터 경과 시간 |
| 상태 피처 | 팀 평균 HP%, 스킬 쿨타임 준비 상태, 플래시 보유율 |
| 이벤트 피처 | 최근 30초 와드 설치/파괴 수, 최근 킬 이벤트 수 |
| 온톨로지 피처 | 현재 구성의 engage_potential score, CC 체인 가능 여부 |
| 메타 피처 | 골드 차이, 킬 차이, 타워 차이, 게임 경과 시간 |

**모델 구조**
- LSTM 기반 시계열 분류기 (30초 슬라이딩 윈도우)
- 출력: 다음 60초 내 한타 발생 확률 P(teamfight | state_t)
- 임계값: P > 0.75 → 관전 포인트 Alert 발생

**한타 감지 로직 (rule-based + ML 앙상블)**
```
RULE 1: 적팀 2인 이상이 1000 유닛 이내 집결 AND 내팀 1인 이상 근접
RULE 2: 오브젝트 스폰 90초 이내 AND 양팀 모두 오브젝트 500 유닛 이내
RULE 3: 10초 내 적 와드 2개 이상 파괴 (시야 장악 시작)
ML: LSTM 확률 > 0.75
→ (RULE 1 OR RULE 2 OR RULE 3) AND ML → 한타 예측 발동
```

### 3-2. 관전 포인트 선정

```
스펙테이터 포인트 우선순위 알고리즘:

1. 전투 중심 좌표(centroid) 계산
2. 오브젝트 위치 가중치 적용
3. 핵심 플레이어 가시성 점수 계산
   - 현재 위치에서 최대한 많은 key player가 보이는 카메라 위치
   - key player = 해당 순간 Flash 보유 + R 쿨타임 완료 + HP > 50%
4. 정보 밀도(information density) = 가시 key player 수 / 전체 플레이어
5. 최고 information density 좌표 → 관전 포인트 확정
```

### 3-2b. pgvectorscale 앙상블 (한타 예측 보강)

```
현재 게임 상태 벡터화
    ↓
pgvectorscale TOP-20 유사 상황 검색
    ↓
해당 상황에서 실제 한타 발생 비율 계산
    → empirical_prob = fights / 20

최종 예측 = 0.5 × LSTM_prob + 0.3 × empirical_prob + 0.2 × rule_score
→ false positive 감소, 패치 변화에 강건
```

### 3-3. 승부 예측 (3단계)

#### Pre-game 예측 (경기 시작 전)
- 입력: 챔피언 구성 벡터(팀A) + 챔피언 구성 벡터(팀B) + 선수 폼 지표 + H2H 기록
- 모델: 구성 벡터 코사인 유사도 기반 K-NN + Gradient Boosting 앙상블
- 출력: 팀A 승리 확률 (ex. 58%)

#### 인게임 Win Probability (실시간)
- 입력: 매 10초 game_state_vector
- 모델: LightGBM (피처: 골드차·킬차·타워차·오브젝트·게임시간)
- 출력: 현재 시점 승률 추이 (선행 연구: 10분 63% → 20분 75% → 30분 83% 정확도)
- pgvectorscale 활용: "지금 상태와 유사했던 100게임에서 A팀 승률"을 베이즈 사전확률로 활용

#### Teamfight Outcome 예측 (한타 직전)
- 입력: 한타 직전 pre_state_vector (HP%, 스킬준비율, 구성 engage_potential, 포지션)
- 모델: 이진 분류기 (Random Forest 또는 XGBoost)
- 출력: "이번 한타에서 팀A가 이길 확률 X%"
- 클라이언트 표시: 한타 시작 시점에 오버레이로 확률 노출

#### 승률 스윙 = 관전 가치 지표
```
teamfight_value = |win_prob_after - win_prob_before|
→ 이 값이 클수록 = 게임 흐름을 바꾼 한타
→ 관전 포인트 우선순위 결정에 재활용
```

### 3-4. 유저 수준별 딥다이브 해설 (Bedrock)

```
Level 1 (Casual):
  - 한타 결과, 주요 킬, 오브젝트 획득 여부
  - "이 한타에서 A팀이 이겼어요. 드래곤을 먹었고 3킬을 챙겼습니다."

Level 2 (Regular):
  - 핵심 CC 체인, 스킬 콤보, 게임체인저 플레이
  - 골드 스윙, 오브젝트 전환 여부

Level 3 (Hardcore):
  - 진입 타이밍의 적절성, 포지셔닝 분석
  - 스킬 순서 최적성, 쿨타임 관리
  - 대안 시나리오 제시

Level 4 (Analyst):
  - 한타 직전 시야 장악도, 와드 위치 최적성
  - 팀 구성의 전투력 발현도
  - 골드/오브젝트 환산 가치, 이후 게임 영향도
  - 유사 프로 경기 사례 비교
```

---

## 통합 아키텍처 (수정판)

```
━━━━━━━━━━━━━━━━ 데이터 수집 레이어 ━━━━━━━━━━━━━━━━

[Riot Esports API]   [spectator-v4 API]   [Live Client Data API]
        │                    │                      │
        └────────────────────┴──────────────────────┘
                             │  (1초 폴링 / webhook)
                             ▼
               [API Gateway + Lambda (Ingestion)]
                             │
                             ▼
                    [Kinesis Data Streams]

━━━━━━━━━━━━━━━━ 저장 레이어 ━━━━━━━━━━━━━━━━

        Kinesis
    ┌───┴───┐
    ▼       ▼
[Lambda  [Lambda
 Parser]  Event Detector]
    │         │
    ▼         ▼
┌────────────────────────────────────────┐
│  TimescaleDB + pgvectorscale           │  ← 핵심 DB (단일 인스턴스)
│  ┌──────────────┐  ┌─────────────────┐ │
│  │ 시계열 테이블  │  │  벡터 테이블    │ │
│  │ player_state │  │ state_embedding │ │
│  │ game_events  │  │ comp_embedding  │ │
│  │ team_state   │  │ fight_embedding │ │
│  │ fight_events │  │ (diskann 인덱스)│ │
│  └──────────────┘  └─────────────────┘ │
└────────────────────────────────────────┘
         │                    │
         ▼                    ▼
   [DynamoDB]          [Neptune]
    현재 스냅샷          정적 온톨로지
    (빠른 조회)          챔피언-아이템-팀-선수

━━━━━━━━━━━━━━━━ 추론 레이어 ━━━━━━━━━━━━━━━━

TimescaleDB + pgvectorscale
    │
    ├─→ [SageMaker: LSTM]          한타 예측
    ├─→ [SageMaker: LightGBM]      Win Probability
    ├─→ [SageMaker: XGBoost]       Teamfight Outcome
    └─→ [pgvectorscale K-NN]       유사 상황 empirical prob

    앙상블 → P(fight) > 0.75 or Win Prob 급변
         │
         ▼
    [EventBridge]

━━━━━━━━━━━━━━━━ 생성 레이어 ━━━━━━━━━━━━━━━━

EventBridge
    │
    ▼
[Lambda: 관전 포인트 계산 + RAG 컨텍스트 구성]
    │  (pgvectorscale로 유사 과거 한타 검색)
    ▼
[Bedrock / Claude API]
 유저 수준별 해설 생성 (Level 1~4)
    │
    ▼
[WebSocket API Gateway]
    │
    ▼
[클라이언트 오버레이/앱]
 - 관전 포인트 좌표
 - Win Probability 실시간 그래프
 - 한타 결과 딥다이브 해설

━━━━━━━━━━━━━━━━ 정적 데이터 파이프라인 ━━━━━━━━━━━━━━━━

[Data Dragon / Riot API]  →  [S3]  →  [Lambda ETL]  →  [Neptune]
                                              │
                                              └→  [SageMaker Feature Store]
```

---

## 데이터 소스

| 소스 | 내용 | 접근 방식 | 용도 |
|------|------|----------|------|
| **Live Stats API** | 1초 단위 전체 플레이어 상태 (위치/HP/쿨타임) | `rfc190Scope: lolesports.aws-usw2-prod.*` | **실시간 운용 핵심** |
| spectator-v5 API | 경기 메타 (챔피언·룬·소환사주문) | developer.riotgames.com | 경기 시작 시 메타 수집 |
| Match-v5 Timeline | 과거 경기 1분 간격 + 이벤트 스트림 | developer.riotgames.com | **학습 데이터셋 구성** |
| Data Dragon | 챔피언·아이템 정적 데이터 | ddragon.leagueoflegends.com | Neptune 정적 온톨로지 |
| lolalytics.com | 카운터·시너지 통계 | 크롤링 (주 1회) | COUNTERS 관계 갱신 |

> ⚠️ 좌표계 주의: Live Stats API는 `{x, z}` 사용. `pos_y` 컬럼은 `pos_z`로 통일.

---

## 참고 연구

- [LoL 실시간 결과 예측 (arXiv, 2023)](https://arxiv.org/pdf/2309.02449)
- [SIDO Performance Model (arXiv, 2024)](https://arxiv.org/pdf/2403.04873)
- [Deep Learning KPI 식별: 승리예측·맵·시야 (ScienceDirect)](https://www.sciencedirect.com/science/article/pii/S2451958825001332)
- [AWS: Rift Rewind - Lambda로 Riot API 연동](https://builder.aws.com/content/34KiyAPCi3wABSmSCEqRzZD0IKC/rift-rewind-querying-riot-games-api-for-match-data-via-aws-lambda)
- [AWS Neptune Knowledge Graph 아키텍처](https://aws.amazon.com/neptune/knowledge-graphs-on-aws/)
- [Riot Live Events API 비공식 문서](https://github.com/SkinSpotlights/LiveEventsDocumentation)

---

## 아키텍처 결정 사항

| 항목 | 결정 | 이유 | 결정일 |
|------|------|------|--------|
| 시계열 DB | TimescaleDB | pgvectorscale 공존, 단일 PostgreSQL 쿼리 | 2026-03-23 |
| 벡터 검색 | pgvectorscale (StreamingDiskANN) | 별도 벡터 DB 불필요, 시계열 JOIN 가능, 23x 속도 | 2026-03-23 |
| TimescaleDB 호스팅 전략 | POC=로컬 Docker → Phase1=EC2 단일 인스턴스 → 상용화=Timescale Cloud | `timescale/timescaledb-ha:pg16-latest` 이미지가 TimescaleDB+pgvectorscale 번들 → 환경 그대로 EC2 이식. RDS/Aurora는 pgvectorscale 미지원으로 제외 | 2026-03-23 |
| 스트림 파이프라인 (Kinesis) | POC/Phase1=생략, 상용화=도입 | POC는 LiveStatsReceiver asyncio 배치 버퍼로 충분. Kinesis는 ML추론·알림 팬아웃 소비자가 생기는 시점에 추가. 로컬 테스트 필요 시 LocalStack 활용 | 2026-03-23 |
| 정적 그래프 | Neptune (openCypher) | 챔피언-아이템-선수 관계 탐색에 그래프 최적 | 2026-03-23 |
| Neptune 동적 변화 관리 | 패치=속성 덮어쓰기+S3 아카이빙 / 이적=관계 타임스탬핑 | 삭제 없이 이력 보존. 경기 중 불변, 경기 사이 업데이트 | 2026-03-23 |
| 스냅샷 | DynamoDB | 현재 게임 상태 빠른 단건 조회 | 2026-03-23 |
| **실시간 데이터 소스** | **Live Stats API** | **1초 단위 위치/HP/쿨타임 제공 확인** | **2026-03-23** |
| 초기 그래프 DB | **Neo4j Desktop** | Neptune 이전 전 로컬 프로토타입. openCypher 동일 → 마이그레이션 불필요 | 2026-03-27 |
| TimescaleDB retention | **player_state만 삭제, teamfight_events 영구 보존** | player_state는 경기당 수만 행, teamfight_events는 pgvectorscale 학습 데이터 핵심 | 2026-03-27 |
| 한타 이력 저장 단위 | **TeamfightRecord 노드 (Neo4j)** | TimescaleDB teamfight_events → 경기 종료 후 Neo4j MERGE. 과거 한타 그래프 탐색 가능 | 2026-03-27 |
| 좌표계 | **{x, z}** | Live Stats API 실제 데이터 샘플 확인 | 2026-03-23 |
| 학습 데이터 소스 | Match-v5 timeline | 공개 API, Challenger 고품질 데이터 수집 가능 | 2026-03-23 |
| 한타 감지 (실시간) | proximity + kill cluster + objective | Live Stats API 1초 스트림 활용 가능 | 2026-03-23 |
| 한타 감지 (학습용) | CHAMPION_KILL 클러스터링 | match-v5 과거 데이터 대상 (1분 간격 한계) | 2026-03-23 |
| 한타 예측 | LSTM + rule-based + K-NN 앙상블 | 각 방식의 단점 보완 | 2026-03-23 |
| 승부 예측 | LightGBM (인게임) + XGBoost (한타) | 해석 가능성 + 속도 | 2026-03-23 |
| 해설 생성 | Bedrock + RAG (pgvectorscale) | deathRecap + 유사 사례 기반 구체적 해설 | 2026-03-23 |

## 다음 단계

- [x] spectator-v4 API 실제 데이터 구조 검증 → [[api-verification]]. v5 업데이트 확인, TRACK A/B 전략 수립 후 **Live Stats API 발견으로 단일 트랙 전환.**
- [x] Neptune 스키마 openCypher 초안 작성 → [[neptune-schema]]. Champion/Ability/Item/Player/Team/MapZone 전체 노드 + 9개 관계 타입 정의.
- [x] TimescaleDB + pgvectorscale 로컬 프로토타입 구성 → [[local-prototype-setup]]. Docker Compose + DDL(pos_z 수정, health/cooldown 컬럼 포함) + 검증 쿼리 완료.
- [x] 챔피언 구성 임베딩 모델 설계 (5챔피언 → 64차원) → [[champion-embedding-design]]. Phase 1 피처 엔지니어링(sum+max+meta 64dim) + Phase 2 DeepSets 설계.
- [x] Match-v5 timeline 기반 학습 데이터 파이프라인 설계 → [[data-pipeline-and-teamfight-detection]]. Challenger 사다리 수집 → 1분 간격 적재 파이프라인 구현.
- [x] 한타 감지 로직 재설계 → [[data-pipeline-and-teamfight-detection]]. CHAMPION_KILL 클러스터링(30s/2000unit/min3kill) + gold_swing + objective 레이블.
- [x] Live Stats API 데이터 구조 분석 및 아키텍처 갱신 → [[live-stats-api-analysis]]. 1초 단위 위치/HP/쿨타임 확인, pos_z 수정, 아키텍처 완전 유효 확인.
- [x] Win Probability LightGBM 학습 실험 → [[win-probability-lightgbm]]. 피처 엔지니어링(gold/kill/tower/dragon/baron/cs_diff), GroupShuffleSplit 경기 단위 분할, 5단계 실험 계획(EXP-001~005, 목표 AUC>0.80), WinProbabilityPredictor 추론 클래스 구현.
- [x] Bedrock 프롬프트 수준별 템플릿 설계 → [[bedrock-prompt-templates]]. L1~L4 시스템 프롬프트 4종, RAG 컨텍스트 빌더(pgvectorscale 유사 한타 검색), deathRecap 텍스트 변환 유틸리티, 실제 해설 예시 포함.
- [x] Live Stats API WebSocket 수신기 구현 → [[live-stats-api-receiver]]. LiveStatsReceiver(asyncio/websockets) + TeamfightDetector(3-rule 앙상블) + 배치 플러시(2초 인터벌) + Bedrock 해설 트리거 파이프라인 설계 완료.
- [x] Esports 데이터 수집 및 Riot ID 매핑 파이프라인 실제 구현 → [[esports-data-scraper]]. lolesports API(getTeams/getSchedule/getEventDetails) + Riot Account API puuid 매핑(MANUAL_OVERRIDES + 다중 태그 전략). 실제 매핑 성공률 86% (98/114명). 미매핑 16명은 players_mapped.csv 수동 입력 가능.
- [x] LCK 2025 경기 데이터(시리즈/게임/픽) 수집 파이프라인 구현 → game_data_pipeline.py. lolesports 스케줄 페이지네이션(older 방향) + window endpoint 픽 데이터. 시리즈 209개 / 게임 685개 / 픽데이터 546개 / 챔피언 138종. --seasons 플래그로 다년도 수집 지원.
- [x] 챔피언 온톨로지 + 패치 이력 수집 파이프라인 구현 → champion_data_pipeline.py. DDragon API 버전별 스탯 수집 + 패치 간 diff로 buff/nerf 자동 분류. 챔피언 172개 / 패치 변화 280건 (버프 136 / 너프 102 / 신규 6). 패치 14.1~15.24 커버.
- [x] 게임별 승패 수집 파이프라인 구현 → game_data_pipeline.collect_game_results(). lolesports live-stats window ending 호출 → totalGold 비교로 승자 판별. 541/555게임 수집(97.5%). API 주의: startingTime 10초 단위 정렬 필수. Game 노드에 winner/loser/winnerSide 속성 저장.
- [x] 선수별 게임 스탯 수집 파이프라인 구현 → game_data_pipeline.collect_player_stats(). ending 시점 window + details API 동시 수집. PlayerGameStats 5,410개 (541게임 × 10선수). 수집 항목: KDA/CS/gold/killParticipation/championDamageShare/wardsPlaced/items/keystoneId/skillOrder/skillMaxOrder. Game 노드에 durationSec/goldDiff/blueDragons/redDragons/barons 추가.
- [x] 스킬 맥스 순서(skillMaxOrder) + 키스톤 승률 분석. "EQWQQRQE..." → "QEW" 변환 로직 구현. LCK 2025 기준: Conqueror 53% / PhaseRush 54% 승률 우위. PressTheAttack 43%로 최저.
- [x] 수집 파이프라인 증분 업데이트 지원. collect_series / collect_game_results / collect_player_stats 모두 캐시 로드 후 신규 항목만 처리하도록 개선. get_all_versions는 항상 최신 조회로 새 패치 자동 감지. cron job으로 매일 신규 경기·패치 자동 수집 가능 (game/champions 단계는 Riot dev key 불필요).
- [x] 실시간 API 연결 시 온톨로지 확장 계획 수립. TeamfightRecord 노드(한타 단위 이력) + PARTICIPATED_IN 관계 추가 예정. TimescaleDB retention: player_state만 삭제 대상, teamfight_events는 영구 보존 → 과거 한타 비교 기능 유지. 실시간-온톨로지 연결 3패턴 정의: A) 한타 감지 시 Neo4j 즉시 조회(CC 콤보·카운터), B) pgvectorscale 유사 한타 → TeamfightRecord Neo4j 조회, C) 경기 종료 후 누적 학습 루프. → neptune-schema.md 섹션 8 추가.



## 2026-04-10 업데이트

- 온톨로지 교육을 체계적으로 계속 받아야 한다는 필요성 재인식. 눈 불편으로 해외선물 실전도 쉬는 상황 → 온톨로지 학습에 집중하기 좋은 시기.
