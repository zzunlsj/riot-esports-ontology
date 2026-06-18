---
type: project-log
date: 2026-05-01
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [progress, timescaledb, win-probability, replay, gate-a, gate-b]
---

# Riot Esports Ontology 진행 상태

## 현재 상태 (2026-04-29)

- **Gate A 완료** ✅ — 전 항목 통과
- TimescaleDB 로컬 컨테이너 `lol-ontology-db` 실행 중
- Match-v5 timeline 2,000 파일 캐시 / 전체 replay 완료 (`team_state` 106,566행 / 2,000경기)
- **최신 모델**: EXP004 LightGBM (AUC 0.8353, 16피처) + EXP-TF-001 XGBoost (Acc 83.4%)
- Neo4j: `CompositionEmbedding` 1,572 노드 적재 완료
- `win_prob_swings.csv` Top 50 생성 + 대표 5경기 선정 완료
- `lolesports_live_poller.py` 실시간 연결 구조 완비 (mock 단계)
- **다음**: Gate B — Bedrock 실 자격증명 + E2E 파이프라인

---

## 모델 버전 이력

| 버전 | 알고리즘 | 피처 수 | AUC | 비고 |
|-----|---------|--------|-----|------|
| EXP001 | LightGBM | 4 | 0.8088 | gold_diff + game_time_sec |
| EXP002 | LightGBM | 6 | 0.8088 | + kill_diff, tower_diff |
| EXP003 | LightGBM | 9 | 0.8424 | baseline 전체, GroupShuffleSplit |
| EXP004 | LightGBM | 16 | **0.8353** | + hp_diff, dmg_share, wards, kill_participation |
| EXP005 | LightGBM | 20 | ≈EXP004 | AUC 미개선, ECE 0.1879→0.0269 (calibration만 개선) |
| EXP006 | LightGBM | 20 | ≈EXP004 | AUC 미개선, calibration 추가 검증 — EXP004 유지 |
| **현재** | **EXP004** | **16** | **0.8353** | `models/win_prob_EXP004.pkl` |

---

## Gate A 완료 검증 (2026-04-29)

### EXP004 In-game Win Prob

- **데이터**: 51,283 frames × 16 피처 (Match-v5 타임라인 2,000경기)
- **피처셋 (baseline 9 + rich 7)**:
  - baseline: `gold_diff`, `kill_diff`, `tower_diff`, `dragon_diff`, `cs_diff`, `level_diff`, `gold_blue_pct`, `gold_diff_norm`, `game_time_min`
  - rich: `hp_diff`, `blue_hp_pct`, `red_hp_pct`, `dmg_share_diff`, `wards_diff`, `ward_kills_diff`, `kill_part_diff`
- **검증**: GroupShuffleSplit (경기 단위, test 20%)
- **결과**: AUC **0.8353** — 컨틴전시 기준(≥0.83) 통과
- **모델 경로**: `scraper/models/win_prob_EXP004.pkl`
- **rich 피처 빌더**: `scraper/scripts/pipeline/exp004_feature_builder.py` → `data/exp004/frames_rich.parquet`

### EXP-TF-001 Teamfight Outcome

- **데이터**: 8,711 한타 (Match-v5 2,000 타임라인에서 클러스터링 추출)
- **클러스터링 파라미터**: WINDOW_SEC=30, RADIUS=2000, MIN_KILLS=3
- **피처 7개**: `gold_diff`, `hp_diff`, `level_diff`, `game_min`, `fight_kills`, `blue_hp_pct`, `red_hp_pct`
- **피처 중요도**: gold_diff(0.285) > level_diff(0.221) > hp_diff(0.152)
- **결과**: Accuracy **83.4%** / AUC **0.91** — Gate 기준(≥60%) 대폭 초과
- **모델 경로**: `scraper/data/exp004/teamfight_outcome_model.ubj`
- **스크립트**: `scraper/scripts/pipeline/exp_tf_001_train.py`

### Composition Embedding

- **데이터**: player_stats.json + game_results.json — 786게임 × 2팀 = 1,572 구성
- **파이프라인**: one-hot (146-dim) → StandardScaler → PCA 64-dim
- **분산 설명률**: 91.3%
- **적재**: Neo4j `CompositionEmbedding` 노드 1,572개
- **PCA 모델 저장**: `scraper/data/exp004/composition_pca.pkl`
- **스크립트**: `scraper/scripts/pipeline/build_composition_embeddings.py`

### Win Prob Swing CSV + 대표 5경기

- **데이터**: team_state 106,566행 / 2,000경기 (replay_cached_matches --all 완료)
- **결과**: `scraper/data/win_prob_swings.csv` (Top 50 swing 모멘트)
- **상세**: `1-projects/riot-esports-ontology/gate-a-completion.md`
- **대표 5경기**:

| match_id | 패턴 | 스윙 크기 | 시간대 |
|----------|------|---------|------|
| KR_7515771175 | 후반 대역전극 | 0.572 | 32~42분 |
| KR_7571663681 | 중반 킬사태 역전 | 0.549 | 14~26분 |
| KR_7570140582 | 타워 롤러코스터 | 0.527 | 21~23분 |
| KR_7568622128 | 초반 판세 포착 | 0.358 | 12~21분 |
| KR_7570336767 | 장기전 오브젝트 | 0.358 | 24~34분 |

---

## 실시간 파이프라인 구조 (Gate A 기준)

### WinProbPredictor (신규)

- `scraper/scripts/realtime/win_prob_predictor.py`
- EXP004(`.ubj`) 우선 로드, 없으면 EXP003(`.pkl`) 폴백
- details endpoint 피처 없을 시 neutral 기본값으로 graceful degradation

### LiveStatsReceiver (업데이트)

- `on_frame()` 시그니처: `async def on_frame(ts_ms, participant_frames, events, details_frame=None)`
- `_predict_win_prob()`: details_frame에서 rich 피처 추출 후 WinProbPredictor 호출

### LolesportsLivePoller (업데이트)

- `_fetch_frames()` 반환: `tuple[list[dict], dict[str, dict]]`
- window + details 병렬 조회, details 실패 시 빈 맵 반환
- `run()` / `replay()` 모두 details_frame 전달 지원
- **실행**: `python -m scripts.realtime.lolesports_live_poller --replay <GAME_ID>`

---

## 운영 커맨드

```bash
cd 1-projects/riot-esports-ontology/scraper
source .venv/bin/activate

# Match-v5 백필
python scripts/pipeline/backfill_matches.py \
  --start-month 2025-01 --end-month 2026-04 \
  --target-per-month 100 --pages 5 --page-size 50

# EXP004 rich 피처 빌드
python -m scripts.pipeline.exp004_feature_builder

# EXP004 학습
python -m scripts.pipeline.exp004_train

# Teamfight Outcome 학습
python -m scripts.pipeline.exp_tf_001_train

# Composition Embedding 빌드 (Neo4j 연결 필요)
set -a && source .env && set +a
python -m scripts.pipeline.build_composition_embeddings

# 전체 캐시 replay (team_state 적재)
python -m scripts.realtime.replay_cached_matches --all

# Swing 분석
python scripts/features/win_prob_swings.py --top 50 --window-size 1

# 라이브 폴러 (실 경기)
export LOLESPORTS_API_KEY=<key>
python -m scripts.realtime.lolesports_live_poller

# 과거 경기 재생 (replay 모드)
python -m scripts.realtime.lolesports_live_poller --replay <GAME_ID>
```

---

## 다음 액션 (Gate B)

- [ ] Bedrock 실 자격증명 연결 + L1~L2 해설 실제 생성 테스트
- [ ] `lolesports_live_poller --replay <GAME_ID>` E2E 완전 처리 (한타 감지 → 해설 생성 로그)
- [ ] WebSocket Alert → 클라이언트 수신 레이턴시 측정 (목표: p95 ≤ 3초)
- [ ] Win Probability 실시간 오버레이 UI 초안
- [ ] EXP-PG-001: Pre-game K-NN (composition_embedding 활용, AUC ≥ 0.70 목표)

---

## 이전 검증 기록 (아카이브)

### replay 중간 진행 이력 (2026-04-22)

- 캐시 timeline 파일 수: 1,950 (최종 2,000)
- 760행 / 12경기 → 2,108행 / 36경기 → 3,488행 / 63경기 → 5,516행 / 103경기 (중간 체크)
- **최종 완료**: 106,566행 / 2,000경기

### EXP003 피처 파일 (2026-04-22)

- `scraper/data/features_winprob.parquet`: rows 39,142 / matches 1,497 / label_mean 0.4889
