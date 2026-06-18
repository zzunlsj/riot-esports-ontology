---
type: research
date: 2026-04-29
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [TIPS, 연구노트, ontology, LightGBM, XGBoost, TimescaleDB, Neo4j, teamfight, win-probability, composition-embedding, gate-a]
---

# TIPS 연구노트 — 이스포츠 실시간 경기 맥락 온톨로지 시스템 구현 현황

**과제명:** 리그 오브 레전드 실시간 경기 맥락 온톨로지 기반 AI 해설·예측 시스템  
**작성일:** 2026-04-29  
**작성자:** RORR (사업전략·상품전략·사업개발 팀장)  
**연구 단계:** Gate A 완료(오프라인 모델 성능 확정) → Gate B 진입(실시간 E2E 파이프라인 MVP)

---

## 1. 연구 배경 및 목표

본 과제는 리그 오브 레전드(LoL) 프로 경기 및 래더 경기 데이터를 실시간/준실시간으로 수집·분석하여, ① 한타 발생 사전 감지, ② 인게임 승률 예측(Win Probability), ③ 유저 수준별 AI 해설 자동 생성을 구현하는 온톨로지 기반 AI 서비스 개발을 목표로 한다.

| 목표 ID | 목표 | 서비스 단계 |
|--------|------|-----------|
| G1 | 한타 발생 전 관전 시점·관전 포인트 자동 포착 | 인게임 실시간 |
| G2 | 한타 종료 후 유저 수준별(L1~L4) 딥다이브 해설 생성 | 한타 직후 |
| G3 | 사전·인게임 승부 예측(Pre-game / Win Prob / Teamfight Outcome) | 사전 + 인게임 |

본 연구노트는 2026-04-22 이후 약 1주간 수행된 **Gate A: 오프라인 모델 성능 확정** 단계의 전 항목 완료를 보고하고, 이어지는 **Gate B: 실시간 E2E 파이프라인 MVP** 단계 진입 계획을 기술한다.

---

## 2. 시스템 아키텍처 개요

본 시스템은 정적 지식 그래프와 동적 시계열/벡터 레이어를 분리한 하이브리드 4-레이어 구조를 유지한다.

```
LAYER 3 : 추론·생성 레이어   — LightGBM(Win Prob) + XGBoost(Teamfight) + Bedrock(Claude API)
LAYER 2B: 벡터 레이어        — pgvectorscale (StreamingDiskANN)
LAYER 2A: 동적 온톨로지      — TimescaleDB (player_state / game_events / team_state)
LAYER 1 : 정적 온톨로지      — Neo4j (챔피언·아이템·팀·선수 그래프 + CompositionEmbedding)
```

**Gate A 단계 기술 변화:**
- LAYER 1 (Neo4j): `CompositionEmbedding` 노드 신규 적재 (1,572개, 64차원 PCA 임베딩)
- LAYER 3: 단일 LightGBM(EXP003)에서 **EXP004 LightGBM(Win Prob) + EXP-TF-001 XGBoost(Teamfight Outcome)** 듀얼 모델 체계로 확장
- 실시간 추론: `WinProbPredictor` 클래스 도입으로 모델 버전 자동 선택 및 폴백 구현

---

## 3. Gate A 완료: 주요 구현 내용 (2026-04-22 ~ 04-29)

### 3-1. EXP004: Match-v5 Rich 피처 기반 Win Probability 모델 고도화

**배경:**  
기존 EXP003 모델(AUC 0.8424)은 `gold_diff`, `kill_diff`, `tower_diff` 등 팀 단위 집계 지표만 사용했다. 그러나 실전에서 골드 격차가 동일해도 체력 상태·데미지 비율·와드 장악 차이에 따라 승률이 크게 달라진다는 점에서, Match-v5 timeline의 세밀한 프레임 데이터를 추가 피처로 활용하는 실험을 설계했다.

**EXP004 Rich 피처 빌더 (`exp004_feature_builder.py`):**

Match-v5 timeline의 `participantFrames`에서 분 단위로 아래 정보를 추출한다.

| 피처 | 추출 소스 | 계산 방법 |
|------|---------|---------|
| `hp_diff`, `blue_hp_pct`, `red_hp_pct` | `championStats.health / healthMax` | 팀 평균 체력 비율 |
| `dmg_share_diff` | `damageStats.totalDamageDoneToChampions` | 팀 총 데미지 대비 팀A 비율 차이 |
| `wards_diff` | `WARD_PLACED` 이벤트 | 와드 설치 수 팀 차이 |
| `ward_kills_diff` | `WARD_KILL` 이벤트 | 와드 파괴 수 팀 차이 |
| `kill_part_diff` | `CHAMPION_KILL` + `assistingParticipantIds` | 킬 참여율 팀 차이 |

**최종 피처셋 (16개):**

```
baseline(9): gold_diff, kill_diff, tower_diff, dragon_diff, cs_diff,
             level_diff, gold_blue_pct, gold_diff_norm, game_time_min
rich(7):     hp_diff, blue_hp_pct, red_hp_pct, dmg_share_diff,
             wards_diff, ward_kills_diff, kill_part_diff
```

**실험 결과:**

| 실험 | 피처 수 | AUC | 학습 데이터 | 비고 |
|-----|--------|-----|-----------|------|
| EXP003 | 9 | 0.8424 | 1,497경기 | 이전 기준점 |
| EXP004 | 16 | **0.8464** | 2,000경기 | ✅ 현재 채택 |
| EXP005 | 20 | 0.8442 | 2,000경기 | 파생 피처 과적합, 롤백 |

- 검증 방법: `GroupShuffleSplit` 경기 단위 분할 (test 20%), 동일 경기 프레임 간 data leakage 완전 차단
- hp_diff 신호 확인: `y=1(블루 승리)` 그룹에서 평균 hp_diff **+4.43%**, `y=0` 그룹에서 **–4.36%** → 체력 차이가 승률과 강한 상관관계를 보임

**EXP005 과적합 원인 분석:**  
`kill_per_min`, `dragon_weight(용 스택 가중)`, `tower_weight(타워 누적 기하), `gold_x_hp` 교차 피처 4개를 추가했으나 AUC가 오히려 하락(0.8464 → 0.8442). num_leaves 확장으로 인한 분산 증가가 교차 피처의 정보 이득을 상쇄한 것으로 판단, EXP004로 롤백.

---

### 3-2. EXP-TF-001: 한타 결과 XGBoost 이진 분류 (신규)

**배경:**  
인게임 Win Probability(분 단위)와 별개로, 한타 발생 직전 상태로 해당 한타의 승패를 예측하는 특화 모델이 필요하다. 한타는 게임 전체가 아닌 국지적 상황(체력·레벨·골드)에 의해 결정되므로, 독립 모델 학습이 적합하다.

**데이터 구성 (`exp_tf_001_train.py`):**  
TimescaleDB가 아닌 **Match-v5 timeline JSON 직접 파싱** 방식을 채택했다. `label_teamfights.py`가 TimescaleDB `game_events` 테이블을 필요로 하는 반면, 이 스크립트는 `data/cache/matches/` 캐시 파일만으로 독립 실행 가능하다.

**클러스터링 파라미터 (기존 방식과 동일):**

| 파라미터 | 값 |
|---------|---|
| 시간 윈도우 | 30초 |
| 공간 반경 | 2,000 유닛 |
| 최소 킬 수 | 3킬 |

**추출 결과:** 2,000 타임라인 파일 → **8,711 한타** (동점 제외)

**피처 (7개):**

| 피처 | 설명 |
|------|------|
| `gold_diff` | 한타 직전 골드 차이 (블루−레드) |
| `hp_diff` | 팀 평균 체력 비율 차이 |
| `level_diff` | 팀 평균 레벨 차이 |
| `game_min` | 한타 발생 시점 (분 단위) |
| `fight_kills` | 한타 총 킬 수 |
| `blue_hp_pct` | 블루팀 평균 체력 비율 |
| `red_hp_pct` | 레드팀 평균 체력 비율 |

**학습 결과:**

| 지표 | 값 |
|------|---|
| Accuracy | **83.4%** |
| AUC | **0.91** |
| Gate 기준 (Accuracy ≥ 60%) | ✅ 대폭 초과 |
| Train / Test | 6,968 / 1,743 한타 |

**피처 중요도:**

| 피처 | 중요도 |
|------|-------|
| gold_diff | 0.285 (1위) |
| level_diff | 0.221 (2위) |
| hp_diff | 0.152 (3위) |
| red_hp_pct | 0.106 |
| blue_hp_pct | 0.095 |
| game_min | 0.078 |
| fight_kills | 0.063 |

**해석:**  
골드 격차가 한타 결과 예측에서도 가장 중요한 단일 변수이지만(0.285), 레벨 차이와 체력 비율이 합산 37.3%로 골드를 상회한다. 이는 "골드는 비슷하지만 레벨 앞서거나 체력이 유리한 팀이 한타를 이긴다"는 실전 관전 감각을 정량적으로 지지하는 결과다.

---

### 3-3. Composition Embedding: 챔피언 구성 벡터화 및 Neo4j 적재 (신규)

**배경:**  
Pre-game Win Prediction(EXP-PG-001) 및 유사 구성 검색 기능의 기반이 되는 챔피언 구성 임베딩을 생성하고 Neo4j에 적재했다. 애초 설계(pgvectorscale의 `composition_embeddings` 테이블)에서, 현재 로컬 프로토타입 단계에서는 Neo4j `CompositionEmbedding` 노드로 먼저 구현하고 향후 pgvectorscale 마이그레이션을 예정하는 실용적 접근을 취했다.

**파이프라인 (`build_composition_embeddings.py`):**

```
player_stats.json + game_results.json
        ↓
  팀별 5챔피언 one-hot 인코딩 (146차원)
        ↓
  StandardScaler (with_std=False, 평균 중심화)
        ↓
  PCA 64차원 (분산 설명률 91.3%)
        ↓
  Neo4j CompositionEmbedding 노드 MERGE
  {game_id, side, champions[], won, winner_side, embedding[64]}
```

**결과:**

| 항목 | 값 |
|------|---|
| 입력 게임 수 | 786경기 |
| 구성 수 (blue + red) | 1,572개 |
| 고유 챔피언 수 | 146개 |
| one-hot 차원 | 146 |
| PCA 출력 차원 | 64 |
| 분산 설명률 | 91.3% |
| Neo4j 노드 수 | 1,572 CompositionEmbedding |
| PCA 모델 저장 | `data/exp004/composition_pca.pkl` |

**설계 주의사항:**  
Neo4j의 Champion 노드에는 `winRate`/`pickRate` 속성이 아직 미적재 상태로, 현재 구현은 픽 패턴(어떤 챔피언을 선택했는가)만 반영한다. 시너지·카운터 등 관계적 정보는 COUNTERS/SYNERGIZES_WITH 엣지로 추후 반영할 예정이다.

---

### 3-4. Win Probability Swing 분석 완료 및 대표 경기 선정

**데이터:**  
`replay_cached_matches.py --all`로 2,000개 timeline 파일을 mock replay 완주하여 `team_state` 51,478행 / 969경기를 적재했다.

**Swing 분석 (`win_prob_swings.py --top 50 --window-size 1`):**  
단위 프레임 간 `win_prob` 절댓값 변화를 기준으로 경기 전체에서 가장 큰 급변 모멘트 50건을 추출한다. 이 도구는 단순 모델 정확도 평가를 넘어 "어느 순간 게임 흐름이 실제로 뒤집혔는가"를 정량화하여 관전 가치 높은 순간을 발굴하는 역할을 한다.

**결과 (`data/win_prob_swings.csv`):**

| match_id | 시간 | 이전→이후 승률 | 스윙 | 패턴 |
|----------|------|-------------|------|------|
| KR_7515771175 | 32분 | 0.785 → 0.213 | **0.572** | 후반 한타 대역전 |
| KR_7571663681 | 26분 | 0.276 → 0.826 | 0.549 | 중반 킬사태 역전 |
| KR_7570140582 | 23분 | 0.222 → 0.749 | 0.527 | 타워 연쇄 롤러코스터 |
| KR_7574335937 | 25분 | 0.361 → 0.853 | 0.492 | 오브젝트 집중 역전 |
| KR_7572922060 | 32분 | 0.484 → 0.051 | 0.433 | 후반 급락 |

**대표 5경기 선정 기준:** 스윙 크기 다양성(0.358~0.572), 역전 방향 다양성(블루↔레드), 발생 시간대 다양성(12~42분), 복수 스윙 여부.

---

### 3-5. 실시간 파이프라인 구조 정비 (Gate B 준비)

실제 경기 연결 전 코드 구조를 정비하여 `lolesports_live_poller.py` → `live_stats_receiver.py` → `win_prob_predictor.py` 체인이 실 경기에 즉시 투입 가능한 상태로 만들었다.

**WinProbPredictor (`win_prob_predictor.py`, 신규):**

```
모델 자동 선택:
  1. EXP004 모델 (data/exp004/model.ubj) 존재 시 우선 로드 → 16피처 모드
  2. 미존재 시 EXP003 모델 (models/win_prob_EXP003.pkl) 폴백 → 9피처 모드
  3. 둘 다 없으면 0.5 반환 (no-model 모드)

graceful degradation:
  - details_frame 없을 때: rich 7 피처 → neutral 기본값 (hp 50%, dmg 0 등)
  - baseline 9 피처만으로 예측 가능하게 설계
```

**LiveStatsReceiver 업데이트:**

`on_frame()` 시그니처에 `details_frame=None` 파라미터를 추가하여 lolesports feed의 `details` 엔드포인트 데이터를 수신할 때 rich 피처를 활성화하도록 했다.

```python
async def on_frame(
    self,
    timestamp_ms: int,
    participant_frames: dict,
    events: list[dict],
    details_frame: dict | None = None   # 추가
) -> None
```

**LolesportsLivePoller 업데이트:**

`_fetch_frames()`가 window 프레임과 details 프레임을 병렬 조회하고, `details` 엔드포인트 실패 시 graceful degradation으로 빈 맵을 반환하도록 구현했다.

```python
async def _fetch_frames(self) -> tuple[list[dict], dict[str, dict]]:
    # window: BASE_FEED/window/{game_id}?startingTime=...
    # details: BASE_FEED/details/{game_id}?startingTime=...
    # details 조회 실패 시 det_map = {} 반환 (window만으로 동작 보장)
```

**details 엔드포인트 구조 (검증 완료):**
- 최상위 `participants[]` 배열, `participantId` 1~10 (blueTeam 1~5, redTeam 6~10)
- 각 참여자에 `currentHealth`, `maxHealth`, `totalDamageDoneToChampions`, `wardsPlaced`, `wardsDestroyed` 포함

---

## 4. 현재 데이터 자산 현황

| 자산 | 규모 / 상태 |
|------|-------------|
| LCK 2025 시리즈 | 209개 |
| 게임 수 | 685개 |
| PlayerGameStats | 5,410개 (541경기 × 10선수) |
| Match-v5 timeline 캐시 | 2,000 파일 |
| 챔피언 온톨로지 | 172개 챔피언, 280건 패치 변화 (버프 136 / 너프 102 / 신규 6) |
| WinProb 학습 피처 | 51,283행 (EXP004, frames_rich.parquet) |
| WinProb 모델 (EXP004) | AUC 0.8464 — `models/win_prob_EXP004.pkl` |
| Teamfight 학습 데이터 | 8,711 한타 (2,000 타임라인에서 클러스터링) |
| Teamfight 모델 (EXP-TF-001) | Acc 83.4% / AUC 0.91 — `data/exp004/teamfight_outcome_model.ubj` |
| Composition Embedding | 1,572 Neo4j 노드 (64차원, PCA 91.3%) |
| team_state 시계열 | 51,478행 / 969경기 |
| Win Prob Swing CSV | Top 50 모멘트 (`data/win_prob_swings.csv`) |

---

## 5. 모델 성능 종합 평가

### 5-1. G3 목표 대비 현황

| 예측 모델 | 성능 목표 | 달성값 | 판정 |
|---------|---------|------|------|
| In-game Win Probability | AUC ≥ 0.85 (컨틴전시 ≥ 0.83) | **0.8464** | ✅ 컨틴전시 기준 통과 |
| Teamfight Outcome | Accuracy ≥ 65% | **83.4%** | ✅ 대폭 초과 |
| Teamfight Outcome AUC | — | **0.91** | ✅ |
| Pre-game Win Pred | AUC ≥ 0.70 | 미착수 | ⏩ Gate B P2 |

### 5-2. 선행 연구 대비 포지셔닝

```
선행 연구 (arXiv:2309.02449, Riot 5만 경기 기반):
  10분 시점 AUC: 0.63 / 20분: 0.75 / 30분: 0.83

본 연구 EXP004 (2,000경기 기반):
  전 시간대 통합 AUC: 0.8464

→ 학습 데이터 규모가 선행 연구의 4%에 불과하지만 HP%·데미지·와드 등 
  풍부한 피처 엔지니어링으로 선행 연구 30분 시점 기준에 근접한 성능 달성
```

### 5-3. 한계 및 고려사항

- **시간대별 AUC 미분석**: 10분/20분/30분 구간별 성능이 측정되지 않음. 실서비스에서 초반 예측 신뢰도를 사용자에게 전달할 때 필요.
- **Brier Score 미측정**: 캘리브레이션(예측 확률 vs 실제 승률 RMSE) 검증 미완료. Platt Scaling 또는 Isotonic Regression 적용 여부 판단 필요.
- **학습 데이터 편향**: Match-v5 기반 Challenger/Master 래더 경기 중심 — LCK 프로 경기 특성 차이가 존재할 수 있음.
- **EXP004 목표 AUC 미달**: 0.85 목표 대비 0.8464 달성. 잔여 0.0036을 위해 시간대별 시간 피처 세분화, 드래곤 영혼 단계 인코딩, 바론 버프 잔여 시간 등을 Gate B 이후에 실험 예정.

---

## 6. 전체 파이프라인 End-to-End 흐름 (현재 상태)

```
[데이터 수집]
  Match-v5 API (Challenger 래더)
       ↓
  data/cache/matches/*_timeline.json (2,000파일)

[피처 엔지니어링]
  exp004_feature_builder.py
       ↓
  data/exp004/frames_rich.parquet (51,283행 × rich 7 피처)
       +
  data/features_winprob.parquet (baseline 9 피처)
       ↓ inner join (match_id, game_time_min)
  학습 데이터 (51,283행 × 16 피처)

[모델 학습]
  exp004_train.py (LightGBM, GroupShuffleSplit)
       → models/win_prob_EXP004.pkl  [AUC 0.8464]

  exp_tf_001_train.py (XGBoost, 8,711 한타)
       → data/exp004/teamfight_outcome_model.ubj  [Acc 83.4%]

  build_composition_embeddings.py (PCA 64차원)
       → Neo4j: 1,572 CompositionEmbedding 노드
       → data/exp004/composition_pca.pkl

[실시간 추론 (준비 완료)]
  lolesports_live_poller.py
       ↓ window + details 병렬 조회
  live_stats_receiver.on_frame(ts_ms, pf, events, details_frame)
       ↓
  win_prob_predictor.predict(...)
       → team_state.win_prob / team_state.model_signal 적재
       → TeamfightDetector 한타 alert

[분석]
  replay_cached_matches --all → team_state (51,478행 / 969경기)
       ↓
  win_prob_swings.py --top 50
       → data/win_prob_swings.csv (급변 Top 50 + 대표 5경기)

[미완료 — Gate B 대상]
  Bedrock 실 자격증명 → L1~L4 해설 생성
  WebSocket → 클라이언트 오버레이
```

---

## 7. Gate B 진입 계획 (2026-05-31 목표)

Gate B의 핵심은 "오프라인 검증 완료된 모델"을 실시간 경기 스트림에 연결하고, 해설 생성까지 E2E 레이턴시를 측정하는 것이다.

### 7-1. P0: Bedrock 실 자격증명 연결 + 해설 생성 검증

현재 `bedrock_commentary.py`는 mock 폴백(하드코딩 해설 문자열)으로 동작한다. Bedrock 실 자격증명을 연결하고 실제 경기 한타 1건에 대해 L1~L4 해설 각각의 레이턴시와 팩트 정확성을 측정한다.

**목표 지표:**
- 해설 생성 레이턴시 ≤ 5초 (한타 종료 감지 → L1 해설 클라이언트 수신)
- Bedrock 월 추정 비용 ≤ $500 (LCK 1시즌 기준)

### 7-2. P0: LoL Esports 실 경기 E2E 처리

`lolesports_live_poller.py --replay <GAME_ID>`로 lolesports feed API의 과거 경기 데이터를 재생하여, 한타 감지 → 해설 생성까지의 완전한 처리 로그를 기록한다.

**제약사항:**  
`feed.lolesports.com` 과거 경기 데이터는 마지막 10프레임(전부 게임 로딩 중 상태, 골드/킬 모두 0)만 저장되어 있어 실질적 분석이 어렵다. 따라서 실 경기 테스트는 라이브 LCK 경기 시 `run()` 모드로 수행하거나, 과거 Match-v5 timeline을 고해상도로 시뮬레이션하는 방향을 병행한다.

### 7-3. P1: WebSocket 레이턴시 측정

Alert 발동 → WebSocket → 클라이언트 수신까지의 왕복 레이턴시를 측정한다.

**목표:** p95 ≤ 3초

### 7-4. P1: Win Probability 실시간 오버레이 UI 초안

`team_state.win_prob` 시계열을 웹 브라우저에서 실시간 그래프로 표시하는 최소한의 오버레이 프로토타입을 구현한다. 기술 스택: Chart.js 또는 Recharts + WebSocket 수신.

### 7-5. P2: EXP-PG-001 Pre-game Win Prediction

Neo4j에 적재된 `CompositionEmbedding` 노드를 활용하여 경기 시작 전 구성 기반 승률 예측을 구현한다.

```
방법: 신규 구성(blue+red) → PCA 변환 → Neo4j K-NN 검색 (코사인 유사도)
     → 유사 구성의 실제 승률 → 경기 시작 전 승률 예측

목표 AUC: ≥ 0.70
```

---

## 8. Gate 로드맵 전체 진행 현황

```
✅ Gate A (오프라인 모델 성능 확정)    완료: 2026-04-29
   ├ EXP004 Win Prob AUC 0.8464      ✅
   ├ EXP-TF-001 Teamfight Acc 83.4%  ✅
   ├ Composition Embedding (1,572)   ✅
   └ Swing CSV + 대표 5경기 선정      ✅

🔜 Gate B (실시간 E2E 파이프라인 MVP)  목표: 2026-06-13
   ├ Bedrock 실 자격증명 + 해설 생성   ❌
   ├ E2E 1경기 완전 처리 로그          ❌
   ├ WebSocket 레이턴시 ≤ 3초          ❌
   └ Win Prob 오버레이 UI 초안          ❌

⬜ Gate C (내부 파일럿 & 검증)          목표: 2026-07-31
⬜ Gate D (외부 파일럿 MVP)             목표: 2026-10-31
⬜ Gate E (플랫폼화 / 수익화 진입)     목표: 2027-Q2+
```

---

## 9. 결론

2026-04-29 기준, Riot Esports 온톨로지 프로젝트의 **Gate A(오프라인 모델 성능 확정) 전 항목을 완료**했다.

핵심 성과는 세 가지다.

첫째, Win Probability 모델을 EXP003(9 피처, AUC 0.8424)에서 EXP004(16 피처, AUC 0.8464)로 고도화했다. HP%, 데미지 비율, 와드, 킬 참여율 등 Match-v5 timeline에서만 추출 가능한 상세 피처를 추가함으로써, 기존 "집계 통계 기반" 모델의 한계를 넘어섰다. 학습 데이터 규모가 선행 연구의 4%에 불과하지만, 피처 엔지니어링으로 선행 연구 30분 시점 수준에 근접했다.

둘째, Teamfight Outcome 모델(EXP-TF-001, XGBoost)이 **Accuracy 83.4% / AUC 0.91**로 Gate 기준(60%)을 대폭 초과하며 학습되었다. 피처 중요도 분석에서 골드 차이(0.285)보다 레벨·체력 합산(0.373)이 더 높게 나타나, 한타 결과를 설명하는 핵심 변수에 대한 실증적 근거가 생겼다.

셋째, 챔피언 구성 임베딩(PCA 64차원, 분산 설명률 91.3%)이 Neo4j에 1,572 노드로 적재되어 Pre-game Win Prediction(EXP-PG-001)과 유사 구성 검색의 데이터 기반이 준비되었다.

이제 "모델이 작동하는가"의 단계는 지났다. 다음 Gate B에서는 이 모델들이 실제 경기 스트림 위에서, Bedrock 해설 생성까지 연결된 E2E 파이프라인으로 **언제, 얼마나 빠르게, 얼마나 정확하게** 동작하는지를 측정한다.

---

## 10. 참고 문헌

- LoL 실시간 결과 예측 (arXiv, 2023) — https://arxiv.org/pdf/2309.02449
- SIDO Performance Model (arXiv, 2024) — https://arxiv.org/pdf/2403.04873
- Deep Learning KPI 식별: 승리예측·맵·시야 (ScienceDirect, 2025)
- AWS Neptune Knowledge Graph 아키텍처 공식 문서
- pgvectorscale StreamingDiskANN 기술 문서 (Timescale)
- XGBoost 공식 문서 (Chen & Guestrin, 2016)

---

## 연결 노트

- [[gate-a-completion]] — Gate A 완료 보고 (결과 수치 상세)
- [[progress-log]] — 최신 수치 / 진행 체크포인트 / 운영 커맨드
- [[goal-metrics-roadmap]] — KPI 목표 및 Gate 기반 로드맵
- [[win-probability-lightgbm]] — Win Probability 실험 상세 (EXP001~EXP005)
- [[data-pipeline-and-teamfight-detection]] — 학습 파이프라인 + 한타 감지 설계
- [[live-stats-api-receiver]] — LiveStatsReceiver 설계
- [[bedrock-prompt-templates]] — L1~L4 해설 프롬프트 설계
- [[champion-embedding-design]] — Composition Embedding 설계 배경
- [[tips-research-note-2026-04-22]] — 이전 연구노트
