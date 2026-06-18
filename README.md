# Riot Esports 실시간 경기 맥락 온톨로지

League of Legends 프로 경기(LCK 등)의 **실시간 맥락**을 온톨로지·지식그래프·ML로 구조화하여,
**한타를 예측하고, 승부를 예측하고, 유저 수준별 딥다이브 해설을 자동 생성**하는 시스템입니다.

> 상태: **active** · Gate A 완료 ✅ → Gate B(실시간 E2E MVP) 진입 준비 중
> 현재 프로토타입 DB: Neo4j Desktop(로컬) + TimescaleDB/pgvectorscale(로컬 Docker) — AWS 이전 전 검증 단계

---

## 프로젝트 목표

| ID | 목표 | 서비스 페이즈 | 현재 단계 |
|----|------|--------------|-----------|
| **G1** | 한타 발생 전 관전 시점·관전 포인트 자동 포착 | In-game | 규칙 엔진 설계 완료, LSTM 모델 미학습 |
| **G2** | 한타 종료 후 유저 수준별(L1~L4) 딥다이브 해설 생성 | Post-fight | Bedrock 프롬프트 설계 완료, 실 자격증명 미연결 |
| **G3** | 사전·인게임 승부 예측 (Pre-game / Win Probability / Teamfight Outcome) | Pre-game + In-game | In-game AUC **0.8353** ✅, Teamfight Acc **83.4%** ✅, Pre-game 미착수 |

---

## 기술 스택

- **데이터 소스**: Riot Esports API · lolesports · Live Stats API · Match-v5 Timeline · Data Dragon
- **정적 지식그래프**: Amazon Neptune (Property Graph, openCypher) — *현재 로컬은 Neo4j Desktop*
- **시계열 + 벡터**: TimescaleDB + pgvectorscale (단일 PostgreSQL 인스턴스)
- **추론/학습**: SageMaker (LSTM · LightGBM · XGBoost) + pgvectorscale K-NN
- **생성**: Amazon Bedrock / Claude API (RAG 기반 수준별 해설)
- **스트리밍/서빙**: Kinesis · Lambda · EventBridge · DynamoDB · WebSocket API Gateway (상용화 단계 도입)

---

## 아키텍처 — 4 레이어

```
┌──────────────────────────────────────────────────────┐
│  LAYER 3 : 추론·생성 레이어                            │  SageMaker + Bedrock
│  한타 예측 / 승부 예측 / 해설 생성 엔진                 │
├──────────────────────────────────────────────────────┤
│  LAYER 2B : 벡터 레이어                                │  pgvectorscale
│  게임 상태 임베딩 · 구성 벡터 · 유사 상황 검색          │
├──────────────────────────────────────────────────────┤
│  LAYER 2A : 동적 온톨로지                              │  TimescaleDB
│  실시간 게임 상태 (시계열) — 동일 DB 인스턴스           │
├──────────────────────────────────────────────────────┤
│  LAYER 1 : 정적 온톨로지                               │  Neptune (Property Graph)
│  챔피언·아이템·팀·선수 지식 그래프                      │
└──────────────────────────────────────────────────────┘
```

**핵심 설계**: Layer 2A(시계열)와 2B(벡터)가 **같은 PostgreSQL 인스턴스** 위에서 동작 → 시계열 쿼리와 벡터 유사도 검색을 **단일 SQL**로 수행. 별도 벡터 DB(Pinecone 등) 불필요.

- **Layer 1 (정적 온톨로지)**: Champion / Ability / Item / Player / Team / MapZone / Objective 노드 + `COUNTERS`, `SYNERGIZES_WITH`, `CHAINS_WITH` 등 9개 관계. 패치마다 갱신, 경기 중에는 불변.
- **Layer 2A (동적 온톨로지)**: `player_state` · `game_events` · `team_state` · `teamfight_events` 시계열 테이블. retention 정책상 `player_state`만 삭제, `teamfight_events`는 학습 자산으로 영구 보존.
- **Layer 2B (벡터)**: 게임 상태 벡터(128dim) · 챔피언 구성 벡터(64dim) · 한타 상태 벡터(128dim). StreamingDiskANN 인덱스.
- **Layer 3 (추론·생성)**: 한타 예측(LSTM + rule + K-NN 앙상블) · Win Probability(LightGBM) · Teamfight Outcome(XGBoost) · Bedrock 수준별 해설(RAG).

> 다이어그램 상세: [architecture-mermaid.md](architecture-mermaid.md) · [architecture-diagram.html](architecture-diagram.html)

---

## 현재 성과 (Gate A 완료, 2026-04-29)

| 산출물 | 결과 |
|--------|------|
| **In-game Win Probability** | LightGBM 16피처 · **AUC 0.8353** (EXP004, GroupShuffleSplit 경기 단위 검증) |
| **Teamfight Outcome** | XGBoost · **Accuracy 83.4% / AUC 0.91** (EXP-TF-001, 한타 8,711건) |
| **Composition Embedding** | one-hot→PCA 64차원(분산 91.3%) · Neo4j `CompositionEmbedding` **1,572 노드** |
| **학습 데이터** | Match-v5 timeline **2,000경기** replay → `team_state` **106,566행** |
| **Swing 분석** | 승률 급변 Top 50 CSV + 대표 5경기 선정 |
| **실시간 파이프라인** | `lolesports_live_poller` 구조 완비 (mock 단계) |

### 모델 버전 이력

| 버전 | 알고리즘 | 피처 | AUC | 비고 |
|------|----------|------|-----|------|
| EXP001 | LightGBM | 4 | 0.8088 | gold_diff + game_time |
| EXP003 | LightGBM | 9 | 0.8424 | baseline 전체 |
| **EXP004** | **LightGBM** | **16** | **0.8353** | **현재 채택** (+HP/데미지/와드/킬참여율) |
| EXP005/006 | LightGBM | 20 | ≈EXP004 | AUC 미개선, calibration만 개선(ECE 0.1879→0.0269) |

---

## 데이터 소스

| 소스 | 내용 | 용도 |
|------|------|------|
| **Live Stats API** | 1초 단위 전체 플레이어 상태(위치/HP/쿨타임) | 실시간 운용 핵심 |
| spectator-v5 API | 경기 메타(챔피언·룬·소환사주문) | 경기 시작 시 메타 수집 |
| Match-v5 Timeline | 과거 경기 1분 간격 + 이벤트 스트림 | 학습 데이터셋 구성 |
| Data Dragon | 챔피언·아이템 정적 데이터 | 정적 온톨로지 |
| lolesports API | 팀·스케줄·픽·승패 데이터 | 경기 온톨로지 적재 |

**데이터 거버넌스**: Riot API / lolesports / Data Dragon만 canonical raw로 취급. 학습셋은 공식 raw + verified derived만 사용.

> ⚠️ 좌표계 주의: Live Stats API는 `{x, z}` 좌표 사용 (`pos_y` 아님).

---

## 로드맵 (Gate 기반)

```
Gate A: 오프라인 모델 성능 확정      ✅ 완료 (2026-04-29)
  ↓
Gate B: 실시간 E2E 파이프라인 MVP    🔜 ~2026-06-13
  ↓
Gate C: 내부 파일럿 & 검증           ~2026-07-31
  ↓
Gate D: 외부 파일럿 MVP              ~2026-10-31
  ↓
Gate E: 플랫폼화 (수익화 진입)        2027-Q2 이후
```

**Gate B 핵심 작업**: Bedrock 실 자격증명 연결 + L1~L2 해설 생성 · 실 경기 1게임 E2E 처리 · Alert→WebSocket 레이턴시 측정(목표 ≤3초) · Win Probability 실시간 오버레이 UI 초안.

> 지표·KPI·Contingency 상세: [goal-metrics-roadmap.md](goal-metrics-roadmap.md)

---

## 문서 가이드

### 핵심
- [_index.md](_index.md) — **기술 아키텍처 전체** (온톨로지 구조, 레이어, 아키텍처 결정 사항)
- [goal-metrics-roadmap.md](goal-metrics-roadmap.md) — 목표지향 KPI 및 Gate 로드맵
- [progress-log.md](progress-log.md) — 진행 상태 상세 로그 및 운영 커맨드
- [service-scenario.md](service-scenario.md) — 서비스 시나리오 및 유저 여정
- [cto-handoff-2026-05-04.md](cto-handoff-2026-05-04.md) — CTO 인수인계 문서

### 설계
- [neptune-schema.md](neptune-schema.md) — Neptune openCypher 스키마
- [champion-embedding-design.md](champion-embedding-design.md) — 챔피언 구성 임베딩(5챔피언→64차원)
- [bedrock-prompt-templates.md](bedrock-prompt-templates.md) — L1~L4 수준별 해설 프롬프트
- [data-pipeline-and-teamfight-detection.md](data-pipeline-and-teamfight-detection.md) — 학습 데이터 파이프라인 + 한타 감지
- [local-prototype-setup.md](local-prototype-setup.md) — TimescaleDB + pgvectorscale 로컬 구성

### 실험·검증
- [win-probability-lightgbm.md](win-probability-lightgbm.md) — Win Prob LightGBM 실험
- [gate-a-completion.md](gate-a-completion.md) — Gate A 완료 보고
- [api-verification.md](api-verification.md) — Riot API 데이터 구조 검증
- [live-stats-api-analysis.md](live-stats-api-analysis.md) · [live-stats-api-receiver.md](live-stats-api-receiver.md) — Live Stats API 분석/수신기
- [implementation-plan-2026-04-11.md](implementation-plan-2026-04-11.md) — 구현 플랜

### 연구노트 / 발표자료
- TIPS 연구노트: [04-13](tips-research-note-2026-04-13.md) · [04-22](tips-research-note-2026-04-22.md) · [04-29](tips-research-note-2026-04-29.md)
- [internal-seminar-deck-2026-04-22.md](internal-seminar-deck-2026-04-22.md) — 내부 세미나 발표

---

## 참고 연구

- [LoL 실시간 결과 예측 (arXiv, 2023)](https://arxiv.org/pdf/2309.02449)
- [SIDO Performance Model (arXiv, 2024)](https://arxiv.org/pdf/2403.04873)
- [Deep Learning KPI 식별: 승리예측·맵·시야 (ScienceDirect)](https://www.sciencedirect.com/science/article/pii/S2451958825001332)
- [AWS Neptune Knowledge Graphs](https://aws.amazon.com/neptune/knowledge-graphs-on-aws/)
- [Riot Live Events API 비공식 문서](https://github.com/SkinSpotlights/LiveEventsDocumentation)
