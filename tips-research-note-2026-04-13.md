---
type: research
date: 2026-04-13
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [TIPS, 연구노트, ontology, LightGBM, TimescaleDB, Neo4j, teamfight]
---

# TIPS 연구노트 — 이스포츠 실시간 경기 맥락 온톨로지 시스템 구현 현황

**과제명:** 리그 오브 레전드 실시간 경기 맥락 온톨로지 기반 AI 해설·예측 시스템  
**작성일:** 2026-04-13  
**작성자:** RORR (사업전략·상품전략·사업개발 팀장)  
**연구 단계:** 로컬 프로토타입 구축 완료 → End-to-End 파이프라인 통합 검증 단계

---

## 1. 연구 배경 및 목표

본 과제는 리그 오브 레전드(LoL) 프로 경기의 실시간 데이터를 수집·분석하여, ① 한타(팀파이트) 발생 사전 감지, ② 실시간 승부 예측(Win Probability), ③ 유저 수준별 AI 해설 자동 생성을 구현하는 온톨로지 기반 AI 서비스 개발을 목표로 한다.

| 목표 | 세부 내용 |
|------|---------|
| 1차 | 한타 발생 전 관전 시점·관전 포인트 자동 포착 |
| 2차 | 한타 종료 후 유저 수준별(L1~L4) 딥다이브 해설 생성 |
| 3차 | 사전·인게임 승부 예측(Pre-game / Win Probability / Teamfight Outcome) |

---

## 2. 시스템 아키텍처 개요

본 시스템은 4개 레이어로 구성된 이종 DB 하이브리드 아키텍처를 채택한다.

```
LAYER 3 : 추론·생성 레이어   — SageMaker(LSTM/LightGBM/XGBoost) + Bedrock(Claude API)
LAYER 2B: 벡터 레이어        — pgvectorscale (StreamingDiskANN 인덱스)
LAYER 2A: 동적 온톨로지      — TimescaleDB (시계열, 1초 단위)
LAYER 1 : 정적 온톨로지      — Neo4j (챔피언·아이템·팀·선수 지식 그래프)
```

**핵심 설계 결정:** TimescaleDB와 pgvectorscale을 동일 PostgreSQL 인스턴스에서 운용함으로써, 시계열 쿼리와 벡터 유사도 검색을 단일 SQL 트랜잭션으로 수행한다. 이는 별도 벡터 DB(Pinecone 등) 도입 없이 근접 이웃 탐색과 시계열 집계를 결합한 쿼리를 가능하게 한다.

---

## 3. 최근 구현 업데이트 (2026-04-11 ~ 04-12)

### 3-1. 학습 데이터 파이프라인 구축 (`collect_matches.py`)

**목적:** Match-v5 API를 통해 Challenger/Grandmaster 사다리 경기의 1분 단위 타임라인 데이터를 수집하고, TimescaleDB의 `player_state` 및 `game_events` 하이퍼테이블에 적재한다.

**주요 구현 내용:**
- Challenger 리그 소환사 ID → puuid 매핑 자동화 (rate limit 준수: 0.06초 인터벌)
- Match-v5 `/matches/{id}/timeline` 엔드포인트를 통해 1분 단위 프레임(participantFrames) + 이벤트 스트림(events) 파싱
- TimescaleDB `execute_batch` 기반 벌크 적재로 I/O 효율화
- 수집 목표: POC 500경기(~2,500한타) → Phase1 10,000경기(~50,000한타)

**기술적 특이사항:**
- Match-v5 timeline 좌표계가 Live Stats API와 동일한 `{x, z}` 좌표계를 사용함을 확인, 스키마 컬럼 `pos_y` → `pos_z` 통일 적용
- `startingTime` 파라미터는 10초 단위 정렬이 필수임을 API 실증 검증을 통해 확인

---

### 3-2. TimescaleDB + pgvectorscale Docker 환경 구축 (`docker-compose.yml` + SQL DDL)

**목적:** `timescale/timescaledb-ha:pg16-latest` 이미지를 활용하여 TimescaleDB(시계열)와 pgvectorscale(벡터 검색)을 단일 컨테이너로 구동하는 로컬 프로토타입 환경을 완성한다.

**DDL 구성 (3단계 순차 초기화):**

| 파일 | 역할 |
|------|------|
| `01_extensions.sql` | `timescaledb`, `vectorscale` 확장 활성화 |
| `02_tables.sql` | `player_state`, `game_events`, `team_state`, `teamfight_events`, `composition_embeddings`, `teamfight_embeddings` 테이블 생성 |
| `03_hypertables.sql` | 시계열 컬럼 기준 하이퍼테이블 변환 + StreamingDiskANN 인덱스 생성 + Retention Policy 설정 |

**최근 수정사항 (fix: primary keys + indexes + retention):**
- 하이퍼테이블 생성 전 Primary Key 누락 오류 수정: `player_state(time, match_id, participant_id)` 복합 PK 추가
- `composition_embeddings` 및 `teamfight_embeddings` 테이블에 `diskann` 인덱스 명시적 추가
- `player_state` 테이블에 90일 Retention Policy 적용 (경기당 수만 행 생성으로 스토리지 최적화 필요)
- `teamfight_events`는 영구 보존 (pgvectorscale 유사 한타 검색의 학습 기반 데이터)

---

### 3-3. 한타 이벤트 클러스터링 기반 레이블링 (`label_teamfights.py`)

**목적:** Match-v5 timeline의 `CHAMPION_KILL` 이벤트를 시공간 클러스터링하여 한타 구간을 자동 레이블링하고, `teamfight_events` 테이블에 적재한다.

**클러스터링 기준:**

| 파라미터 | 값 | 근거 |
|---------|---|------|
| 시간 윈도우 | 30초 | 단일 한타의 전형적 지속 시간 |
| 공간 반경 | 2,000 유닛 | 팀파이트 교전 범위 |
| 최소 킬 수 | 3킬 이상 | 솔로킬과 팀파이트 구분 |

**생성 레이블:**
- `fight_id` (UUID), 전투 중심 좌표(`centroid_x/z`), `gold_swing`, `kills_in_fight`
- 한타 직후 획득 오브젝트 (`objective_converted`) — 한타 가치 평가 지표

**최근 수정사항 (fix: data validation guards):**
- 레이블 커버리지 경고 추가: 전체 경기 중 한타 레이블이 부족할 경우(`coverage < 5%`) 로그 출력
- `gold_swing` 계산 시 timeline 프레임 경계 오류(IndexError) 방어 코드 추가

---

### 3-4. Win Probability LightGBM 학습 파이프라인 (`train_win_prob.py`)

**목적:** TimescaleDB에 적재된 학습 데이터를 기반으로 LightGBM 이진 분류 모델을 학습하여 인게임 실시간 승률을 예측한다.

**피처 엔지니어링 (`game_state_features.py`):**

| 피처 그룹 | 피처 목록 |
|----------|---------|
| 골드 | `gold_diff` (팀A-팀B 누적 골드 차이) |
| 킬 | `kill_diff` (킬 수 차이) |
| 타워 | `tower_diff` (파괴 타워 수 차이) |
| 오브젝트 | `dragon_diff`, `baron_diff` |
| CS | `cs_diff` (팀 전체 CS 차이) |
| 시간 | `game_time_norm` (정규화 경과 시간) |

**실험 계획 (EXP-001 ~ EXP-005):**

| 실험 ID | 피처셋 | 목표 AUC |
|---------|-------|---------|
| EXP-001 | gold_diff 단일 | 베이스라인 |
| EXP-002 | 골드+킬+타워 | > 0.75 |
| EXP-003 | EXP-002 + 오브젝트 | > 0.78 |
| EXP-004 | EXP-003 + CS+시간 | > 0.80 |
| EXP-005 | EXP-004 + pgvectorscale empirical prior | > 0.82 |

**경기 단위 데이터 분할:** `GroupShuffleSplit`을 사용하여 동일 경기 내 프레임이 train/test에 혼재하는 데이터 누출(leakage)을 방지. 경기 ID 기준으로 train 80% / test 20% 분리.

**추론 클래스 (`WinProbabilityPredictor`):**
- `predict_proba(game_state_dict) -> float` 인터페이스로 실시간 추론 캡슐화
- 모델 직렬화: `joblib` 기반 `.pkl` 저장, 핫 로딩 지원

---

### 3-5. Live Stats API WebSocket 수신기 (`live_stats_receiver.py`)

**목적:** Riot Live Stats API WebSocket 스트림을 asyncio 기반으로 구독하여 1초 단위 플레이어 상태를 TimescaleDB에 실시간 적재하고, 한타 감지 시 Bedrock 해설 파이프라인을 트리거한다.

**핵심 구성요소:**

| 컴포넌트 | 역할 |
|---------|------|
| `LiveStatsReceiver` | asyncio/websockets 기반 스트림 수신기, 2초 배치 플러시 |
| `TeamfightDetector` | 3-rule 앙상블 기반 실시간 한타 감지 |
| Bedrock 트리거 | 한타 감지 시 L1~L4 해설 생성 요청 |

**한타 감지 규칙 (Rule-based + ML 앙상블):**

```
RULE 1: 상대팀 2인 이상이 1,000 유닛 이내 집결 AND 아군 1인 이상 근접
RULE 2: 오브젝트 스폰 90초 이내 AND 양팀 모두 오브젝트 500 유닛 이내
RULE 3: 10초 내 적 와드 2개 이상 파괴 (시야 장악 시작 신호)
ML   : LightGBM Win Probability 급변 (Δ > 5%p / 10초)
→ (RULE 1 OR RULE 2 OR RULE 3) AND ML → 한타 예측 발동 (임계값 P > 0.75)
```

**최근 수정사항 (refactor: constants + type safety):**
- 하드코딩된 임계값을 모두 상수(`PROXIMITY_THRESHOLD`, `OBJECTIVE_TIMER_WINDOW`, `WARD_DESTRUCTION_COUNT`)로 추출
- `participant_id` 타입 혼재(int/str) 방어 처리 추가

---

### 3-6. Bedrock L1~L4 수준별 해설 생성 (`bedrock_commentary.py`)

**목적:** 한타 감지 직후 pgvectorscale로 유사 과거 한타를 검색하고, 그 컨텍스트를 RAG 기반으로 Claude API에 전달하여 유저 수준별 해설을 생성한다.

| 수준 | 대상 | 해설 내용 |
|------|------|---------|
| L1 (Casual) | 일반 시청자 | 한타 결과, 주요 킬, 오브젝트 획득 |
| L2 (Regular) | 준숙련 유저 | CC 체인, 게임체인저 플레이, 골드 스윙 |
| L3 (Hardcore) | 숙련 유저 | 진입 타이밍, 스킬 최적성, 대안 시나리오 |
| L4 (Analyst) | 분석가 수준 | 시야 장악도, 팀 구성 전투력 발현도, 유사 프로 경기 비교 |

**RAG 파이프라인:**
1. 한타 직전 `pre_state_vector` (128차원) → pgvectorscale StreamingDiskANN 검색
2. 유사 과거 한타 TOP-5 `outcome`, `gold_swing`, `objective_converted` 추출
3. 시스템 프롬프트(수준별) + 게임 상태 요약 + RAG 컨텍스트 조합 → Claude API 호출
4. 생성된 해설 → WebSocket API Gateway를 통해 클라이언트 오버레이 전달

---

## 4. 현재 데이터 자산 현황

| 자산 | 규모 |
|------|------|
| LCK 2025 시리즈 | 209개 |
| 게임 수 | 685개 (승패 수집 97.5%, 541게임) |
| PlayerGameStats | 5,410개 (541게임 × 10선수) |
| 챔피언 온톨로지 | 172개 챔피언, 280건 패치 변화 (버프 136 / 너프 102 / 신규 6) |
| 패치 커버리지 | 14.1 ~ 15.24 |
| 선수 puuid 매핑 | 86% (98/114명 자동 매핑, 16명 수동 보완 대상) |

---

## 5. 미완료 항목 및 향후 계획

| 항목 | 상태 | 비고 |
|------|------|------|
| Neo4j 수집 데이터 실제 적재 | ❌ 미착수 | `run_pipeline.py --step neo4j` 실행 필요 |
| TimescaleDB Docker 실제 구축 | ❌ 미착수 | `docker-compose up` + DDL 초기화 |
| Match-v5 학습 데이터 수집 (500경기) | ❌ 미착수 | `collect_matches.py` 실행 |
| 한타 레이블링 (`label_teamfights.py`) | ❌ 미착수 | 수집 완료 후 |
| Win Probability LightGBM 학습 (EXP-001~005) | ❌ 미착수 | 학습 데이터 적재 후 |
| 챔피언 구성 임베딩 생성 + pgvectorscale 저장 | ❌ 미착수 | `champion_embeddings.py` |
| Live Stats API 수신기 실제 연결 테스트 | ❌ 미착수 | 실시간 경기 필요 |
| Bedrock 연동 실제 테스트 | ❌ 미착수 | AWS 자격증명 설정 필요 |

---

## 6. 기술적 차별점 요약

1. **단일 PostgreSQL 인스턴스에서 시계열 + 벡터 검색 통합**: TimescaleDB + pgvectorscale 공존으로 외부 벡터 DB 불필요. pgvector 대비 약 23배 빠른 ANN 검색(StreamingDiskANN).

2. **CHAMPION_KILL 클러스터링 기반 한타 레이블링 자동화**: Match-v5 이벤트 스트림에서 시공간 클러스터링(30초/2,000유닛/최소 3킬)으로 한타 구간을 자동 추출하여 수동 레이블링 비용을 제거.

3. **pgvectorscale empirical prior를 활용한 Win Probability 앙상블**: 유사 과거 게임 상황 TOP-K 검색 결과를 베이즈 사전확률로 활용, LightGBM 예측 안정성 향상.

4. **유저 수준 적응형 해설 (L1~L4)**: 단일 모델이 아닌 수준별 시스템 프롬프트 + RAG 컨텍스트로 동일 이벤트에 대해 다층적 해설을 생성하여 이스포츠 콘텐츠 개인화를 실현.

---

## 7. 참고 문헌

- LoL 실시간 결과 예측 (arXiv, 2023) — https://arxiv.org/pdf/2309.02449
- SIDO Performance Model (arXiv, 2024) — https://arxiv.org/pdf/2403.04873
- Deep Learning KPI 식별: 승리예측·맵·시야 (ScienceDirect, 2025)
- AWS Neptune Knowledge Graph 아키텍처 공식 문서
- Riot Live Events API 비공식 문서 (SkinSpotlights/LiveEventsDocumentation)

---

## 연결 노트

- [[implementation-plan-2026-04-11]] — End-to-End 구현 플랜 (체크박스 기반)
- [[data-pipeline-and-teamfight-detection]] — 학습 파이프라인 + 한타 감지 상세
- [[win-probability-lightgbm]] — Win Probability 실험 계획
- [[live-stats-api-receiver]] — WebSocket 수신기 설계
- [[neptune-schema]] — Neo4j 정적 온톨로지 스키마
- [[local-prototype-setup]] — TimescaleDB Docker 구성
