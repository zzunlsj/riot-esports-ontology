---
type: project
date: 2026-05-04
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [handoff, CTO, design-doc, gate-b, architecture]
---

# CTO 인수인계 문서 — Riot Esports 실시간 경기 맥락 온톨로지

> **작성일**: 2026-05-04
> **작성자**: RORR (사업전략·상품전략·사업개발 팀장)
> **수신자**: CTO
> **목적**: 로컬 프로토타입 재현 + Gate B 구현 연계
> **현재 상태**: Gate A 완료 → Gate B 진입 준비 중

---

## 1. 프로젝트 개요

### 무엇을 만드는가

리그 오브 레전드(LoL) 프로 경기를 실시간으로 분석하여 세 가지를 제공하는 AI 서비스:

| 목표 | 설명 | 현재 단계 |
|------|------|---------|
| G1: 한타 포착 | 한타 발생 전 관전 시점 자동 감지 | 규칙 엔진 완료, ML 미착수 |
| G2: 맞춤 해설 | 유저 수준별(L1~L4) AI 해설 자동 생성 | 프롬프트 설계 완료, Bedrock 미연결 |
| G3: 승부 예측 | 사전/인게임 승률 예측 | Win Prob AUC 0.8353 완료 |

### 서비스 흐름 요약

```
실시간 경기 데이터
    → 한타 감지 (TeamfightDetector)
    → 승률 예측 업데이트 (WinProbPredictor)
    → 맞춤 해설 생성 (Bedrock Claude)
    → WebSocket → 클라이언트 오버레이
```

---

## 2. 시스템 아키텍처

### 2-1. 4-레이어 구조

```
LAYER 3 : 추론·생성     — LightGBM(Win Prob) + XGBoost(Teamfight) + AWS Bedrock(Claude)
LAYER 2B: 벡터           — pgvectorscale (StreamingDiskANN)
LAYER 2A: 동적 온톨로지  — TimescaleDB (player_state / game_events / team_state)
LAYER 1 : 정적 온톨로지  — Neo4j (챔피언·아이템·팀·선수 그래프 + CompositionEmbedding)
```

### 2-2. 현재(로컬) vs 목표(클라우드) 구성

| 구성 요소 | 현재 (로컬) | 목표 (Gate D 이후) |
|---------|-----------|----------------|
| 그래프 DB | Neo4j Desktop | **Neo4j AuraDB** (관리형 클라우드) |
| 시계열 DB | TimescaleDB Docker | Timescale Cloud 또는 EC2 |
| 벡터 검색 | pgvectorscale Docker | pgvectorscale EC2 |
| 해설 생성 | mock 폴백 | AWS Bedrock (Claude) |
| 실시간 파이프라인 | 로컬 Python | EC2 + AWS Kinesis (Gate D+) |
| 클라이언트 | 없음 | WebSocket 오버레이 |

**Gate B 핵심**: 로컬 파이프라인에서 AWS Bedrock만 원격 호출하는 구조로 충분. 전체 클라우드 이전은 Gate D 과제.

### 2-3. 그래프 DB — Amazon Neptune이 아닌 Neo4j AuraDB를 선택하는 이유

Amazon Neptune이 AWS 네이티브 그래프 DB로 자연스럽게 보이지만, 이 프로젝트에서는 **Neo4j AuraDB**가 옳은 선택입니다.

**① 마이그레이션 비용 제로**
현재 코드베이스 전체가 `neo4j-python-driver`(Bolt 프로토콜)로 작성되어 있습니다. Neptune은 Bolt를 지원하지 않고 HTTPS/WebSocket 기반 OpenCypher를 사용합니다. Neptune으로 이전하면 `neo4j_client.py`를 포함한 모든 DB 클라이언트 코드를 재작성해야 합니다. 반면 Neo4j AuraDB는 Bolt 호환 — `.env`의 `NEO4J_URI`만 바꾸면 됩니다.

**② CompositionEmbedding 자산 보존**
Gate A에서 구축한 1,572개 CompositionEmbedding 노드와 K-NN 쿼리 로직이 Neo4j 위에 완성된 상태입니다. Neptune으로 이전하면 이 작업을 처음부터 다시 해야 합니다.

**③ Graph Data Science (GDS) 라이브러리**
Gate C에서 적용할 MiroFish 선수 에이전트 모델링은 **복잡한 그래프 순회와 관계 분석**이 필요합니다 — 선수 간 시너지 관계, 팀 행동 군집 탐지, 챔피언 친화도 경로 분석 등. Neo4j GDS는 65개 이상의 그래프 알고리즘(커뮤니티 탐지, 중심성, 경로 탐색 등)을 제공하지만 Neptune은 이에 상응하는 알고리즘 라이브러리가 없습니다.

**④ 비용 구조**
Neptune은 인스턴스 최소 비용이 $0.10/시간 = 약 $73/월(사용 여부 무관). Neo4j AuraDB Free 티어는 Gate C 프로토타입 규모를 커버하며, 유료 티어도 $65/월부터 시작합니다.

**⑤ 벤더 락인 회피**
Neptune은 AWS 독점 Gremlin/OpenCypher 혼합 방언을 사용해 이전 시 재작성이 필요합니다. Neo4j는 오픈소스 기반으로 온프레미스 ↔ AuraDB 이동이 자유롭습니다.

> `neptune-schema.md`는 초기 설계 탐색 문서로 보관하되, 실제 구현 참조 문서에서는 제외합니다.

---

## 3. 기술 스택

```
Language:    Python 3.11
Graph DB:    Neo4j 5.x (Desktop → AuraDB, bolt://localhost:7687)
Time-series: TimescaleDB 2.x (Docker, PostgreSQL 16 기반)
Vector:      pgvectorscale (vectorscale extension)
ML:          LightGBM, XGBoost, scikit-learn
Async:       asyncio, websockets, aiohttp
AI:          AWS Bedrock (anthropic.claude-3-5-sonnet-20241022-v2:0)
GraphRAG:    neo4j-graphrag-python (Text2CypherRetriever, Gate C~)
SDK:         boto3, neo4j-python-driver, psycopg2-binary, neo4j-graphrag
```

---

## 4. 로컬 환경 재현 가이드

### 4-1. 사전 요구사항

```bash
# Python 가상환경
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Docker (TimescaleDB)
docker-compose up -d  # scraper/docker-compose.yml 사용

# Neo4j Desktop 설치 + 로컬 DB 생성
# https://neo4j.com/download/
```

### 4-2. 환경변수 설정 (scraper/.env)

```
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=yourpassword

TIMESCALE_HOST=localhost
TIMESCALE_PORT=5432
TIMESCALE_DB=lol_ontology
TIMESCALE_USER=postgres
TIMESCALE_PASSWORD=yourpassword

RIOT_API_KEY=RGAPI-xxxxxxxx

# Gate B에서 추가:
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=us-east-1
BEDROCK_MODEL_ID=anthropic.claude-3-5-sonnet-20241022-v2:0
```

### 4-3. DB 초기화 순서

```bash
# TimescaleDB 스키마 초기화
psql -h localhost -U postgres -d lol_ontology -f sql/init/01_extensions.sql
psql -h localhost -U postgres -d lol_ontology -f sql/init/02_tables.sql
psql -h localhost -U postgres -d lol_ontology -f sql/init/03_hypertables.sql

# Neo4j 데이터 적재
python run_pipeline.py --step neo4j
python run_pipeline.py --step champions
python run_pipeline.py --step games
python run_pipeline.py --step game-results
python run_pipeline.py --step player-stats
```

### 4-4. 데이터 수집 및 모델 학습 재현 순서

```bash
# 1. LCK 2025 데이터 수집
python scripts/pipeline/collect_matches.py

# 2. Match-v5 timeline 수집 (2,000경기)
python scripts/pipeline/backfill_matches.py

# 3. Win Prob 피처 생성
python scripts/features/game_state_features.py      # baseline 9피처
python scripts/features/exp004_feature_builder.py   # rich 7피처 추가

# 4. 모델 학습
python scripts/models/train_win_prob.py             # EXP004 LightGBM
python scripts/models/exp_tf_001_train.py           # Teamfight XGBoost

# 5. Composition Embedding 생성
python scripts/features/build_composition_embeddings.py  # Neo4j 1,572 노드

# 6. 전체 캐시 mock 재생 (파이프라인 검증)
python scripts/realtime/replay_cached_matches.py --all

# 7. Win Prob 급변 분석
python scripts/features/win_prob_swings.py --top 50
```

**재현 완료 기준:**
- Neo4j: Player 114명, Champion 172개, Game 685개, PlayerGameStats 5,410개
- TimescaleDB: team_state 106,566행 / 2,000경기
- 모델: win_prob_EXP004.pkl (AUC 0.8353), teamfight_outcome_model.ubj (Acc 83.4%)

---

## 5. 현재 데이터 자산

| 자산 | 규모 |
|------|------|
| LCK 2025 시리즈 | 209개 |
| 게임 수 | 685개 |
| PlayerGameStats | 5,410개 |
| Match-v5 timeline 캐시 | 2,000 파일 (data/cache/matches/) |
| 챔피언 온톨로지 | 172개, 280건 패치 변화 |
| team_state 시계열 | 106,566행 / 2,000경기 |
| Teamfight 학습 데이터 | 8,711 한타 |
| CompositionEmbedding | 1,572 Neo4j 노드 (PCA 64차원, 분산 91.3%) |
| Win Prob Swing CSV | Top 50 모멘트 (data/win_prob_swings.csv) |

---

## 6. 모델 성능 현황 (Gate A 확정값)

| 모델 | 파일 | 지표 | 값 |
|------|------|------|---|
| Win Probability (EXP004) | models/win_prob_EXP004.pkl | AUC | 0.8353 |
| Teamfight Outcome (EXP-TF-001) | data/exp004/teamfight_outcome_model.ubj | Accuracy / AUC | 83.4% / 0.91 |
| Composition PCA | data/exp004/composition_pca.pkl | 분산 설명률 | 91.3% |

**EXP004 피처 (16개):**

```
baseline(9): gold_diff, kill_diff, tower_diff, dragon_diff, cs_diff,
             level_diff, gold_blue_pct, gold_diff_norm, game_time_min

rich(7):     hp_diff, blue_hp_pct, red_hp_pct, dmg_share_diff,
             wards_diff, ward_kills_diff, kill_part_diff
```

**Teamfight 피처 중요도:**
- gold_diff: 0.285 (1위)
- level_diff: 0.221 (2위)
- hp_diff: 0.152 (3위)

---

## 7. 파일 구조 (핵심 파일)

```
scraper/
├── .env                               # 환경변수 (재현 시 직접 생성)
├── docker-compose.yml                 # TimescaleDB + pgvectorscale
├── requirements.txt
├── run_pipeline.py                    # Neo4j 데이터 적재 엔트리포인트
├── neo4j_client.py                    # Neo4j 연결 + CRUD
├── test_and_verify.py                 # 환경 검증
├── scripts/
│   ├── pipeline/
│   │   ├── collect_matches.py         # LCK 2025 데이터 수집
│   │   ├── backfill_matches.py        # Match-v5 timeline 수집
│   │   └── label_teamfights.py        # 한타 클러스터링 레이블
│   ├── features/
│   │   ├── game_state_features.py     # Win Prob baseline 피처
│   │   ├── exp004_feature_builder.py  # Win Prob rich 피처 (Match-v5)
│   │   ├── build_composition_embeddings.py
│   │   └── win_prob_swings.py
│   ├── models/
│   │   ├── train_win_prob.py          # LightGBM 학습 (EXP004)
│   │   ├── exp_tf_001_train.py        # XGBoost 한타 분류 학습
│   │   └── win_prob_predictor.py      # 추론 클래스 (모델 자동 선택)
│   ├── realtime/
│   │   ├── lolesports_live_poller.py  # LoL Esports feed 폴링
│   │   ├── live_stats_receiver.py     # on_frame() 수신기
│   │   ├── teamfight_detector.py      # 룰 기반 한타 감지
│   │   └── replay_cached_matches.py   # 캐시 경기 mock 재생
│   └── commentary/
│       ├── bedrock_commentary.py      # Bedrock 해설 생성 (현재 mock)
│       └── rag_context_builder.py     # RAG 컨텍스트 구성
├── sql/
│   └── init/
│       ├── 01_extensions.sql          # timescaledb, vectorscale 활성화
│       ├── 02_tables.sql              # player_state, game_events, team_state
│       └── 03_hypertables.sql         # 하이퍼테이블 변환
├── models/
│   └── win_prob_EXP004.pkl
└── data/
    ├── cache/matches/                 # Match-v5 timeline JSON (2,000파일)
    ├── exp004/
    │   ├── frames_rich.parquet        # 51,283행 × 16피처
    │   ├── teamfight_outcome_model.ubj
    │   └── composition_pca.pkl
    └── win_prob_swings.csv
```

---

## 8. Gate B 구현 가이드 (목표: 2026-06-13)

> Gate B 핵심 목표: 오프라인 검증된 모델을 실제 경기 스트림에 연결하고 E2E 레이턴시 측정

### P0-1: AWS Bedrock 연결 (예상 1일)

**AWS 사전 설정:**

1. AWS 콘솔 → Amazon Bedrock → Model access 메뉴
2. `anthropic.claude-3-5-sonnet-20241022-v2:0` 모델 접근 활성화 (리전: us-east-1 권장)
3. IAM 사용자 생성, 아래 권한 정책 부여:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "bedrock:InvokeModel",
      "bedrock:InvokeModelWithResponseStream"
    ],
    "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.claude-*"
  }]
}
```

4. Access Key / Secret Key 발급 → scraper/.env에 주입

**코드 수정 포인트:**
- `scripts/commentary/bedrock_commentary.py`: mock 폴백 제거, 실 boto3 호출로 교체
- `InvokeModelWithResponseStream` 사용 권장 (레이턴시 체감 개선)

**Gate B 검증 지표:**
- L1 해설 생성 레이턴시 ≤ 5초 (한타 감지 → 클라이언트 수신)
- Bedrock 월 추정 비용 ≤ $500 (LCK 1시즌 기준)

**모델 선택 권장:**
- L1/L2: claude-haiku (속도 + 비용 최적화)
- L3/L4: claude-sonnet (품질 우선)

---

### P0-2: E2E 1경기 완전 처리 로그 (예상 5일)

```bash
# 라이브 경기 없을 경우: Match-v5 고해상도 시뮬레이션
python scripts/realtime/replay_cached_matches.py --game-id KR_7515771175 --verbose

# 라이브 경기 있을 경우:
python scripts/realtime/lolesports_live_poller.py --run
```

**확인할 로그 체인:**
```
[POLLER]     frame received: ts=1234567
[RECEIVER]   on_frame: win_prob=0.623
[DETECTOR]   teamfight detected: fight_id=TF_001
[COMMENTARY] L1 generated: "블루팀이 바론 앞 한타에서..."
[DB]         team_state inserted: win_prob=0.623, model_signal=TF_001
```

**제약 주의**: lolesports feed API 과거 경기는 마지막 10프레임(게임 로딩 상태)만 유효. 실 테스트는 라이브 LCK 경기 대기 또는 Match-v5 고해상도 시뮬레이션으로 대체 가능.

---

### P1-1: WebSocket 레이턴시 측정 (예상 2일)

신규 파일 `scripts/realtime/websocket_server.py` 구현:
- Alert 발동 → WebSocket broadcast → 클라이언트 수신까지 왕복 시간 측정
- 목표: p95 ≤ 3초

---

### P1-2: Win Probability 실시간 오버레이 UI (예상 5일)

- team_state.win_prob 시계열 → Chart.js 또는 Recharts
- WebSocket으로 1초마다 클라이언트에 win_prob 전송
- 블루/레드팀 Win Prob 대비 곡선 실시간 표시

---

### P2-1: EXP-PG-001 Pre-game Win Prediction (예상 3일)

Neo4j CompositionEmbedding K-NN 검색:
1. 신규 구성(blue+red 5챔피언) → composition_pca.pkl로 PCA 64차원 변환
2. Neo4j 코사인 유사도 K-NN → 유사 구성 Top-K 조회
3. 유사 구성들의 실제 승률 평균 → Pre-game 예측값

목표 AUC: ≥ 0.70

---

### P2-2: player_state 개인 신호 보존 (권장 추가)

Gate C에서 선수 행동 임베딩 추출 시 필요한 데이터를 지금 저장해두는 것. Gate B에서 details_frame을 이미 수신하므로 비용 낮음.

```sql
-- sql/init/02_tables.sql에 추가
ALTER TABLE player_state
  ADD COLUMN per_player_hp_pct JSONB,      -- {"1": 0.82, "2": 0.45, ...}
  ADD COLUMN per_player_dmg_share JSONB,   -- 팀 총 딜 대비 개인 비율
  ADD COLUMN per_player_fight_part JSONB;  -- 킬 참여율
```

---

## 9. AI 서비스 레이어 — 두 가지 흐름

이 프로젝트의 AI 서비스는 **성격이 다른 두 흐름**으로 완전히 분리됩니다. 혼동하면 아키텍처가 복잡해집니다.

```
[Flow A] 실시간 해설 — 이벤트 기반, 단방향
  한타 발생 이벤트
      → RAG 컨텍스트 수집 (Neo4j 정적 쿼리)
      → Bedrock Claude (Haiku/Sonnet)
      → L1~L4 서사 텍스트 생성
      → WebSocket → 클라이언트
  NeoConverse: 불필요
  구현 시점: Gate B

[Flow B] 쿼리 인터페이스 — 사용자 질의, 양방향
  사용자 자연어 질문
      → neo4j-graphrag-python Text2CypherRetriever
      → Bedrock Claude (Haiku, 스키마 가이드)
      → Neo4j Cypher 실행
      → Bedrock Claude (Sonnet, 답변 생성)
      → 자연어 응답
  NeoConverse: 불필요 (neo4j-graphrag-python이 대체)
  구현 시점: Gate C
```

### 9-1. L1~L4 해설 수준 정의 (Flow A)

| 레벨 | 대상 | 핵심 내용 | 예상 길이 | 권장 모델 |
|------|------|---------|---------|---------|
| L1 | 관전 입문 | 결과 요약 + 쉬운 설명 | 2~3문장 | Haiku |
| L2 | 일반 시청자 | 핵심 플레이 + 게임 영향 | 5~7문장 | Haiku |
| L3 | 하드코어 팬 | 스킬 분석 + 타이밍 판단 | 10~15문장 | Sonnet |
| L4 | 분석가/프로 지망 | 전략적 맥락 + 대안 시나리오 | 구조화 보고서 | Sonnet |

프롬프트 템플릿 상세: [[bedrock-prompt-templates]]

### 9-2. 쿼리 인터페이스 구현 방식 (Flow B)

**NeoConverse는 프로덕션에서 사용하지 않습니다.** Neo4j Labs 실험 프로젝트로, SSO·엔터프라이즈 암호화·가드레일 등이 미지원입니다. 동일한 기능을 `neo4j-graphrag-python` 공식 패키지로 구현합니다.

팀 구현 부담:

```python
# scripts/query/analyst_query.py
# 팀이 작성해야 하는 전부: 예시 10개 + 래퍼 ~60줄

from neo4j_graphrag.retrievers import Text2CypherRetriever
from neo4j_graphrag.llm import LLMInterface
import boto3, json

class BedrockLLM(LLMInterface):
    """Bedrock Claude를 neo4j-graphrag에 연결하는 30줄 래퍼"""
    def __init__(self):
        self.client = boto3.client("bedrock-runtime", region_name="us-east-1")

    def invoke(self, input_: str) -> str:
        resp = self.client.invoke_model(
            modelId="anthropic.claude-haiku-4-5-20251001",
            body=json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": 500,
                "messages": [{"role": "user", "content": input_}]
            })
        )
        return json.loads(resp["body"].read())["content"][0]["text"]

# 팀이 작성하는 핵심: 도메인 예시 10개
EXAMPLES = [
    ("T1 선수 목록은?",
     "MATCH (p:Player)-[:PLAYS_FOR]->(t:Team {name:'T1'}) RETURN p.name, p.role"),
    ("평균 킬이 가장 높은 선수 TOP 5",
     "MATCH (p:Player)-[:HAD_STATS]->(s:PlayerGameStats) RETURN p.name, avg(s.kills) AS avgK ORDER BY avgK DESC LIMIT 5"),
    # ... 8개 더
]

retriever = Text2CypherRetriever(
    driver=neo4j_driver,
    llm=BedrockLLM(),
    examples=EXAMPLES      # Claude가 이 예시를 few-shot으로 참고해 Cypher 생성
)

# 사용
answer = retriever.search("공격성 높은 정글러가 선호하는 챔피언 승률은?")
```

**Gate C에서 MiroFish 선수 에이전트 그래프 완성 후** 예시를 선수 행동 속성 기반으로 확장:

```python
# Gate C 추가 예시
("aggression이 높은 선수는?",
 "MATCH (p:Player) WHERE p.aggression > 0.7 RETURN p.name, p.role, p.aggression ORDER BY p.aggression DESC"),
("골드 열세에서 한타를 자주 강제하는 팀은?",
 "MATCH (p:Player)-[:PLAYS_FOR]->(t:Team) WHERE p.fight_seek_rate > 0.6 RETURN t.name, avg(p.fight_seek_rate) AS teamAggression ORDER BY teamAggression DESC"),
```

### 9-3. NeoConverse 포지셔닝

| 용도 | 결정 |
|------|------|
| 프로덕션 서비스 | 사용 안함 |
| Gate C 내부 탐색 | 선택적 사용 가능 (어떤 질의 패턴이 자주 나오는지 파악용) |
| Gate D 이후 | neo4j-graphrag-python 커스텀 구현으로 대체 |

> NeoConverse를 Gate C 내부에서 탐색 도구로 쓴다면: 어떤 질문이 자주 나오는지 → 그 패턴만 EXAMPLES에 추가 → neo4j-graphrag-python에 이식하는 흐름으로 활용합니다.

---

## 10. Gate 로드맵 전체

```
Gate A (오프라인 모델 성능 확정)     완료: 2026-04-29
  Win Prob AUC 0.8353 / Teamfight Acc 83.4% / CompositionEmbedding 1,572 노드

Gate B (실시간 E2E 파이프라인 MVP)   목표: 2026-06-13
  Bedrock 연결 / E2E 로그 / WebSocket p95 ≤3초 / Win Prob 오버레이 UI

Gate C (내부 파일럿 & 검증 + MiroFish + 쿼리 인터페이스)  목표: 2026-07-31
  [Flow A] Alert Precision ≥65% / 해설 팩트 정확성 ≥90% / 웹 오버레이 MVP
  [Flow A] MiroFish 선수 에이전트 → CompositionEmbedding 고도화 / EXP-TF-002
  [Flow B] neo4j-graphrag-python + Bedrock Text2Cypher 구현
           예시 10개 작성 → 애널리스트 쿼리 인터페이스 내부 검증

Gate D (외부 파일럿 MVP)              목표: 2026-10-31
  EC2 이전 (Neo4j AuraDB 포함) / LSTM 한타 예측 고도화
  [Flow A] L3/L4 RAG 연결 (pgvectorscale 유사 한타 컨텍스트)
  [Flow B] 애널리스트·코치 쿼리 인터페이스 외부 공개
           가상 로스터 구성 / Monte Carlo 매치 시뮬레이터 MVP

Gate E (플랫폼화 / 수익화 진입)       목표: 2027-Q2+
  Freemium (L1/L2 무료) / 구독 (L3/L4 + 쿼리 인터페이스)
  B2B 방송사 / 팀·코치 라이선스 (Flow B 기반)
```

---

## 11. Gate C 핵심 확장 — MiroFish 선수 에이전트 모델링

### 왜 Gate C인가

Gate B가 완료되면 세 가지 조건이 동시에 충족됩니다:

1. **데이터 준비 완료**: Gate B의 `per_player_hp_pct`, `per_player_dmg_share`, `per_player_fight_part` 개인 신호가 누적됨
2. **파이프라인 검증 완료**: 실시간 데이터 흐름이 안정화된 상태에서 새 레이어를 추가하는 것이 안전
3. **가장 큰 구조적 공백 해소**: 현재 시스템의 가장 약한 링크가 Player 노드의 행동 표현 부재

Gate B 이전에 설계하면 아직 존재하지 않는 데이터를 가정해야 합니다. Gate C 이후로 미루면 Teamfight 모델과 Pre-game 예측이 Gate C 내부 파일럿 기간 내내 선수 맹점을 안고 가게 됩니다.

---

### 무엇을 바꾸는가 — 선수 노드의 패러다임 전환

**현재 Player 노드:**
```
(:Player) → 통계 컨테이너
  {name, team, role, playerGameStats[]}
  → "어떤 결과를 냈는가"만 표현
```

**Gate C 이후 Player 노드 (에이전트):**
```
(:Player) → 행동 에이전트
  {name, team, role,
   aggression: float,           # 선제 교전 시도 빈도
   fight_seek_rate: float,       # 골드 열세 시 한타 강제 비율
   deficit_response: enum,       # "split" | "force_fight" | "stall"
   champion_affinity: vector,    # 선수-챔피언 행동 친화 임베딩
   profile_source: enum,         # "pro" | "ladder" | "prior_only"
   profile_confidence: float}    # 0.0~1.0, 데이터 누적에 따라 상승
```

이 전환은 MiroFish의 핵심 개념과 동일합니다: 에이전트는 단순한 데이터 레코드가 아니라 **성격(personality)·기억(memory)·행동 논리(behavioral logic)**를 가진 독립 개체입니다. LoL 선수 10명이 한 경기를 만들 듯, 시뮬레이션에서도 10개의 에이전트가 상호작용하면 경기가 창발됩니다.

---

### Gate C 구현 작업 목록

**P0: 행동 속성 추출 파이프라인**

```bash
# Gate B에서 누적된 per_player_* 데이터 기반 추출
python scripts/agents/extract_player_behavioral_attrs.py

# 출력: Neo4j Player 노드에 aggression, fight_seek_rate, deficit_response 속성 업데이트
# 기반 데이터: TimescaleDB player_state (per_player_fight_part, per_player_hp_pct)
#             PlayerGameStats 5,410개 (kills, deaths, assists, goldEarned)
#             teamfight_events 8,711건 (한타 참여 패턴)
```

**P0: Player-Champion 행동 임베딩 (CompositionEmbedding 고도화)**

현재 CompositionEmbedding은 "어떤 챔피언을 픽했는가"만 인코딩합니다. 여기에 선수 행동 벡터를 결합하면 "이 선수가 이 챔피언을 플레이할 때"의 임베딩이 됩니다.

```python
# 기존
composition_embedding = champion_onehot_pca(64dim)

# Gate C 이후
composition_embedding = concat(
    champion_onehot_pca(64dim),     # 기존 유지
    player_behavioral_vector(16dim)  # 신규 추가
)  # → 80차원, Pre-game 예측 AUC 향상 기대
```

**P1: Teamfight 모델 피처 보강 (EXP-TF-002)**

EXP-TF-001의 7개 피처에 선수 행동 속성 3개 추가:
- `blue_aggression_score`: 블루팀 5선수 aggression 합산
- `aggression_mismatch`: 양 팀 공격성 편차 (높을수록 한타 발생 예측 신뢰도 상승)
- `deficit_response_blue`: 블루팀 골드 열세 시 한타 강제 성향

목표: EXP-TF-001 Accuracy 83.4% → 86%+

**P1: 맞춤 해설 선수 DNA 주입 (Bedrock 프롬프트 개선)**

현재 해설은 게임 수치 중심입니다. Gate C에서 선수 행동 프로파일을 Bedrock 컨텍스트에 추가합니다:

```python
# bedrock_commentary.py 프롬프트 컨텍스트 추가
player_context = {
    "player_name": "Gumayusi",
    "aggression": 0.78,  # 상위 12%
    "historical_pattern": "골드 열세 상황에서 한타 강제 비율 71%",
    "champion": "Jinx (fight_seek_rate와 높은 친화도)"
}
```

**P2: 신규 선수 Cold-start 전략**

| 케이스 | 대응 |
|--------|------|
| A: 래더 출신 루키 | Match-v5 PUUID 조회 → 래더 데이터로 속성 추출 → `profile_source: "ladder"`, `confidence: 0.4` |
| B: 해외 이적 선수 | 역할 평균 prior + 챔피언 풀 prior → `profile_source: "prior_only"`, `confidence: 0.2` → 경기 누적으로 rapid update |

Bayesian update 스케줄:
```
경기 1~3:   Prior 80% + 관측 20%
경기 4~10:  Prior 50% + 관측 50%
경기 11+:   Prior 20% + 관측 80% → "pro" 전환
```

**P3: Monte Carlo 매치 시뮬레이터 프로토타입**

선수 에이전트가 갖춰지면 가상 매치 시뮬레이션이 가능해집니다:

```python
def simulate_match(team_a_agents, team_b_agents, n_trials=1000):
    results = []
    for _ in range(n_trials):
        game_state = initialize_from_composition(team_a, team_b)
        for decision_point in generate_decision_timeline():
            action = vote_by_agents(agents, game_state)  # 선수 에이전트 행동 투표
            if should_fight(action, game_state):
                outcome = teamfight_model.predict(game_state)  # EXP-TF-001/002
                game_state = update(outcome)
        results.append(game_state.winner)
    return SimulationResult(win_rate=mean(results), trajectory=...)
```

지원하는 시나리오:
- 기존 팀 vs 기존 팀 대진 시뮬레이션
- 가상 로스터 구성 (선수를 자유롭게 조합) + 시뮬레이션
- "역대 드림팀" — behavioral embedding이 있으면 시대를 초월한 가상 대결 가능

---

## 연결 문서

- [[tips-research-note-2026-04-29]] — Gate A 완료 전체 상세
- [[goal-metrics-roadmap]] — KPI 목표 및 Gate 기반 로드맵
- [[implementation-plan-2026-04-11]] — Phase별 구현 플랜 (Gate A 완료)
- [[bedrock-prompt-templates]] — L1~L4 프롬프트 설계
- [[architecture-mermaid]] — 아키텍처 다이어그램
- [[neptune-schema]] — 초기 탐색 문서 (참조 전용, 구현 대상 아님)
