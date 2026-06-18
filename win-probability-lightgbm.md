---
type: research
date: 2026-04-30
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [LightGBM, win-probability, ML, training, feature-engineering]
---

# Win Probability LightGBM 학습 실험

> 인게임 실시간 승률 예측 모델 설계 및 학습 실험 계획.
> 입력: 매 10초(또는 1분) 단위 game_state_vector → 출력: P(팀A 승리)

---

## 1. 문제 정의

### 예측 타겟

```
y = 1  (해당 경기에서 블루팀 승리)
y = 0  (레드팀 승리)

이진 분류 → predict_proba()[:, 1] = P(blue_wins | state_t)
```

### 예측 시점

- **학습 기준**: 경기 종료 후 match-v5 timeline 기반 (1분 간격 스냅샷)
- **추론 기준**: Live Stats API 1초 스트림 → 10초마다 예측 갱신
- **레이블**: 경기 최종 결과 (winning_team = 100 → y=1)

---

## 2. 피처 엔지니어링

### 2-1. 피처 그룹

```python
# scripts/features/game_state_features.py

FEATURE_GROUPS = {

    # ── 골드 피처 ──────────────────────────────────────
    "gold_diff": "total_gold_blue - total_gold_red",        # 절대 차
    "gold_diff_norm": "gold_diff / (game_time_sec + 1)",    # 시간 정규화
    "gold_blue_pct": "total_gold_blue / (total_gold_blue + total_gold_red)",

    # ── 킬 피처 ────────────────────────────────────────
    "kill_diff": "kills_blue - kills_red",
    "kill_diff_norm": "kill_diff / (game_time_sec / 60 + 1)",

    # ── 타워 피처 ──────────────────────────────────────
    "tower_diff": "towers_blue - towers_red",
    "inhibitor_diff": "inhibitors_blue - inhibitors_red",

    # ── 오브젝트 피처 ───────────────────────────────────
    "dragon_diff": "dragons_blue - dragons_red",
    "baron_diff": "barons_blue - barons_red",
    "dragon_soul": "1 if dragon_soul_secured else 0",       # 드래곤 영혼

    # ── 게임 시간 피처 ──────────────────────────────────
    "game_time_sec": "경기 경과 시간 (초)",
    "game_phase": "0(early<15min) | 1(mid 15-25) | 2(late>25)",

    # ── CS 피처 ────────────────────────────────────────
    "cs_diff": "total_cs_blue - total_cs_red",
    "jungle_cs_diff": "jungle_cs_blue - jungle_cs_red",

    # ── 구성 피처 (composition embedding 활용) ──────────
    "comp_sim_blue": "cosine_sim(comp_vec_blue, historical_win_centroid)",
    "comp_advantage": "comp_score_blue - comp_score_red",   # Neptune 온톨로지 활용

    # ── 한타 직후 피처 (live 데이터 전용) ──────────────
    "recent_kill_balance": "kills_blue_last30s - kills_red_last30s",
    "hp_advantage_blue": "avg_hp_pct_blue - avg_hp_pct_red",    # Live Stats API
    "flash_advantage_blue": "flash_ready_blue - flash_ready_red", # 플래시 준비 수
    "ult_ready_blue": "r_ready_count_blue / 5.0",                 # R 준비율
    "ult_ready_red": "r_ready_count_red / 5.0",
}
```

### 2-2. 피처 계산 파이프라인

```python
# scripts/features/build_feature_matrix.py
"""
match-v5 timeline → (X, y) 피처 행렬 생성
1분 간격 스냅샷 → 각 snapshot을 독립 학습 샘플로 취급
"""
import numpy as np
import pandas as pd
import psycopg2

DB_CONN = "postgresql://loluser:lolpass@localhost:5432/lol_ontology"


def extract_features_for_match(match_id: str, conn) -> list[dict]:
    """
    경기 1개 → 분 단위 피처 행렬 행 목록 반환.
    match-v5 timeline player_state + game_events 조합.
    """
    cur = conn.cursor()

    # 분 단위 팀 골드 집계 (TimescaleDB time_bucket 활용)
    cur.execute("""
        SELECT
            time_bucket('60 seconds', time) AS bucket,
            SUM(CASE WHEN team_id = 100 THEN total_gold ELSE 0 END) AS gold_blue,
            SUM(CASE WHEN team_id = 200 THEN total_gold ELSE 0 END) AS gold_red,
            SUM(CASE WHEN team_id = 100 THEN level ELSE 0 END) AS level_blue,
            SUM(CASE WHEN team_id = 200 THEN level ELSE 0 END) AS level_red,
            SUM(CASE WHEN team_id = 100 THEN minions_killed + jungle_cs ELSE 0 END) AS cs_blue,
            SUM(CASE WHEN team_id = 200 THEN minions_killed + jungle_cs ELSE 0 END) AS cs_red
        FROM player_state
        WHERE match_id = %s
        GROUP BY bucket
        ORDER BY bucket
    """, (match_id,))
    gold_rows = cur.fetchall()

    # 킬 이벤트 집계 (누적)
    cur.execute("""
        SELECT
            time_bucket('60 seconds', time) AS bucket,
            SUM(CASE WHEN source_participant_id <= 5 THEN 1 ELSE 0 END) AS kills_blue,
            SUM(CASE WHEN source_participant_id > 5 THEN 1 ELSE 0 END) AS kills_red
        FROM game_events
        WHERE match_id = %s
          AND event_type = 'CHAMPION_KILL'
        GROUP BY bucket
        ORDER BY bucket
    """, (match_id,))
    kill_rows = {r[0]: r for r in cur.fetchall()}

    # 오브젝트 이벤트 집계 (누적)
    cur.execute("""
        SELECT timestamp_ms / 60000 AS minute,
               monster_type, building_type, source_participant_id
        FROM game_events
        WHERE match_id = %s
          AND event_type IN ('ELITE_MONSTER_KILL', 'BUILDING_KILL')
        ORDER BY timestamp_ms
    """, (match_id,))
    obj_rows = cur.fetchall()

    # 경기 결과 조회
    cur.execute("""
        SELECT source_participant_id
        FROM game_events
        WHERE match_id = %s AND event_type = 'GAME_END'
        LIMIT 1
    """, (match_id,))
    result = cur.fetchone()
    # game_end event에서 winning team 파악: source_participant_id <= 5 → blue wins
    # (실제 구현에서는 match-v5 metadata의 win 컬럼 사용)

    rows = []
    cum_kills_blue = cum_kills_red = 0
    cum_towers_blue = cum_towers_red = 0
    cum_dragons_blue = cum_dragons_red = 0
    cum_barons_blue = cum_barons_red = 0

    for i, (bucket, g_blue, g_red, lv_blue, lv_red, cs_blue, cs_red) in enumerate(gold_rows):
        game_sec = i * 60

        # 킬 누적
        if bucket in kill_rows:
            _, kb, kr = kill_rows[bucket]
            cum_kills_blue += (kb or 0)
            cum_kills_red += (kr or 0)

        # 오브젝트 누적 (해당 분까지)
        for (min_obj, monster_type, building_type, src_pid) in obj_rows:
            if min_obj > i:
                break
            is_blue = (src_pid or 0) <= 5
            if monster_type in ('DRAGON', 'EARTH_DRAGON', 'FIRE_DRAGON',
                                 'WATER_DRAGON', 'AIR_DRAGON', 'HEXTECH_DRAGON',
                                 'CHEMTECH_DRAGON'):
                if is_blue: cum_dragons_blue += 1
                else: cum_dragons_red += 1
            elif monster_type == 'BARON_NASHOR':
                if is_blue: cum_barons_blue += 1
                else: cum_barons_red += 1
            elif building_type in ('TOWER_BUILDING', 'INNER_TURRET'):
                if is_blue: cum_towers_blue += 1
                else: cum_towers_red += 1

        # 피처 계산
        gold_diff = (g_blue or 0) - (g_red or 0)
        game_phase = 0 if game_sec < 900 else (1 if game_sec < 1500 else 2)

        row = {
            "match_id": match_id,
            "game_time_sec": game_sec,
            "game_phase": game_phase,
            "gold_diff": gold_diff,
            "gold_diff_norm": gold_diff / (game_sec + 1),
            "gold_blue_pct": (g_blue or 0) / max((g_blue or 0) + (g_red or 0), 1),
            "kill_diff": cum_kills_blue - cum_kills_red,
            "tower_diff": cum_towers_blue - cum_towers_red,
            "dragon_diff": cum_dragons_blue - cum_dragons_red,
            "baron_diff": cum_barons_blue - cum_barons_red,
            "level_diff": (lv_blue or 0) - (lv_red or 0),
            "cs_diff": (cs_blue or 0) - (cs_red or 0),
        }
        rows.append(row)

    return rows


def build_dataset(match_ids: list[str], conn) -> pd.DataFrame:
    all_rows = []
    for mid in match_ids:
        rows = extract_features_for_match(mid, conn)
        all_rows.extend(rows)

    df = pd.DataFrame(all_rows)
    return df
```

---

## 3. 모델 학습

### 3-1. LightGBM 설정

```python
# scripts/train/train_win_probability.py
"""
Win Probability 모델 학습:
- 입력: 분 단위 게임 상태 피처 (match-v5 기반)
- 출력: P(blue_team_win | state_t) ∈ [0, 1]
- 평가: AUC-ROC, Brier Score, Calibration Curve
"""
import lightgbm as lgb
import numpy as np
import pandas as pd
from sklearn.model_selection import GroupShuffleSplit
from sklearn.metrics import roc_auc_score, brier_score_loss
from sklearn.calibration import calibration_curve
import matplotlib.pyplot as plt


FEATURE_COLS = [
    "game_time_sec", "game_phase",
    "gold_diff", "gold_diff_norm", "gold_blue_pct",
    "kill_diff", "tower_diff",
    "dragon_diff", "baron_diff",
    "level_diff", "cs_diff",
]

TARGET_COL = "blue_win"
GROUP_COL = "match_id"   # 같은 경기의 데이터가 train/test에 섞이지 않도록


def train(df: pd.DataFrame):
    X = df[FEATURE_COLS]
    y = df[TARGET_COL]
    groups = df[GROUP_COL]

    # 경기 단위 train/test 분할 (data leakage 방지)
    gss = GroupShuffleSplit(n_splits=1, test_size=0.2, random_state=42)
    train_idx, test_idx = next(gss.split(X, y, groups))

    X_train, X_test = X.iloc[train_idx], X.iloc[test_idx]
    y_train, y_test = y.iloc[train_idx], y.iloc[test_idx]

    # LightGBM 파라미터
    params = {
        "objective": "binary",
        "metric": ["auc", "binary_logloss"],
        "learning_rate": 0.05,
        "num_leaves": 63,
        "max_depth": 6,
        "min_child_samples": 20,
        "feature_fraction": 0.8,
        "bagging_fraction": 0.8,
        "bagging_freq": 5,
        "lambda_l1": 0.1,
        "lambda_l2": 0.1,
        "verbose": -1,
    }

    dtrain = lgb.Dataset(X_train, label=y_train)
    dval = lgb.Dataset(X_test, label=y_test, reference=dtrain)

    callbacks = [
        lgb.early_stopping(50),
        lgb.log_evaluation(100),
    ]

    model = lgb.train(
        params,
        dtrain,
        num_boost_round=2000,
        valid_sets=[dtrain, dval],
        valid_names=["train", "val"],
        callbacks=callbacks,
    )

    return model, X_test, y_test


def evaluate(model, X_test, y_test):
    y_pred = model.predict(X_test)

    auc = roc_auc_score(y_test, y_pred)
    brier = brier_score_loss(y_test, y_pred)

    print(f"AUC-ROC : {auc:.4f}")
    print(f"Brier   : {brier:.4f}  (낮을수록 캘리브레이션 ↑)")

    # 캘리브레이션 곡선 (예측 확률 신뢰도 검증)
    prob_true, prob_pred = calibration_curve(y_test, y_pred, n_bins=10)
    plt.figure(figsize=(6, 6))
    plt.plot(prob_pred, prob_true, marker='o', label='LightGBM')
    plt.plot([0, 1], [0, 1], 'k--', label='Perfect')
    plt.xlabel("Predicted Probability")
    plt.ylabel("True Probability")
    plt.title("Calibration Curve — Win Probability Model")
    plt.legend()
    plt.tight_layout()
    plt.savefig("outputs/calibration_curve.png")

    return {"auc": auc, "brier": brier}


def plot_feature_importance(model, feature_names):
    importance = pd.DataFrame({
        "feature": feature_names,
        "gain": model.feature_importance(importance_type="gain"),
        "split": model.feature_importance(importance_type="split"),
    }).sort_values("gain", ascending=False)

    print("\n피처 중요도 (gain 기준):")
    print(importance.to_string(index=False))
    return importance
```

---

## 4. 실험 결과 (완료)

### 4-1. 실험 이력

| 실험 ID | 피처셋 | 실제 AUC | 데이터 | 상태 |
|---------|-------|---------|-------|------|
| EXP001 | gold_diff, game_time_sec (4개) | 0.8088 | 1,497경기 | 완료 |
| EXP002 | + kill_diff, tower_diff (6개) | 0.8088 | 1,497경기 | 완료 |
| EXP003 | baseline 9개 (dragon/baron/cs/level 추가) | **0.8424** | 1,497경기 | 완료 |
| EXP004 | baseline 9 + rich 7 (hp/dmg_share/wards/kill_part) | **0.8353** | 2,000경기 | ✅ 현재 채택 |
| EXP005 | EXP004 + 파생 피처 4개 (kill_per_min, weighted_dragon 등) | ≈EXP004 | 2,000경기 | AUC 미개선, ECE 0.1879→0.0269 (calibration 개선만) |
| EXP006 | EXP005 기반 추가 검증 | ≈EXP004 | 2,000경기 | AUC 미개선, EXP004 유지 확정 |

> **채택 모델**: EXP004 — `scraper/models/win_prob_EXP004.pkl`

### 4-2. EXP004 상세

```
데이터셋:
  - Match-v5 타임라인 2,000경기 (Challenger/Master)
  - exp004_feature_builder.py → frames_rich.parquet (51,283 rows)
  - features_winprob.parquet (inner join: game_time_min 기준)
  - 최종 학습 행 수: 51,283 frames × 16 피처

피처셋 (16개):
  baseline(9): gold_diff, kill_diff, tower_diff, dragon_diff, cs_diff,
               level_diff, gold_blue_pct, gold_diff_norm, game_time_min
  rich(7):     hp_diff, blue_hp_pct, red_hp_pct, dmg_share_diff,
               wards_diff, ward_kills_diff, kill_part_diff

rich 피처 출처 (exp004_feature_builder.py):
  - hp_pct: championStats.health / championStats.healthMax (1분 스냅샷)
  - dmg_share: damageStats.totalDamageDoneToChampions 집계
  - wards_diff: WARD_PLACED 이벤트 팀별 차이
  - ward_kills_diff: WARD_KILL 이벤트 팀별 차이
  - kill_part_diff: 킬 참여율 (kills + assists) / max(team_kills, 1)

검증: GroupShuffleSplit (경기 단위, test 20%)
결과: AUC 0.8353 (컨틴전시 기준 ≥0.83 통과)
```

### 4-3. 시간대별 성능 (EXP003 기준, EXP004 미측정)

```python
# 게임 시간대별 AUC 분석 (선행 연구: 시간 흐를수록 예측 정확도 ↑)
def evaluate_by_phase(model, df_test):
    for phase_name, phase_range in [
        ("10분 이전", (0, 600)),
        ("10-20분", (600, 1200)),
        ("20-30분", (1200, 1800)),
        ("30분 이후", (1800, 9999)),
    ]:
        mask = (df_test["game_time_sec"] >= phase_range[0]) & \
               (df_test["game_time_sec"] < phase_range[1])
        subset = df_test[mask]
        if len(subset) < 100:
            continue
        auc = roc_auc_score(subset["blue_win"], model.predict(subset[FEATURE_COLS]))
        print(f"{phase_name:12s}: AUC = {auc:.4f}  (n={len(subset):,})")
```

### 4-4. Phase 2 목표 — 피처 확장 (10,000경기 이상)

> Gate B 이후 계획. composition embedding / COUNTERS 관계 활용 등 추가 피처 검증.

```
목표: AUC > 0.85
추가 피처 후보:
  - composition embedding (64dim PCA → 8dim 압축)
  - 한타 직후 상태 (gold_swing, kill_balance_last_fight)
  - 구성 카운터 스코어 (Neo4j COUNTERS 관계 활용, Gate B)
  - 드래곤 영혼 + 바론 버프 잔여 시간
```

### 4-5. 시간대별 성능 분석

```python
# 게임 시간대별 AUC 분석 (선행 연구: 시간 흐를수록 예측 정확도 ↑)
def evaluate_by_phase(model, df_test):
    for phase_name, phase_range in [
        ("10분 이전", (0, 600)),
        ("10-20분", (600, 1200)),
        ("20-30분", (1200, 1800)),
        ("30분 이후", (1800, 9999)),
    ]:
        mask = (df_test["game_time_sec"] >= phase_range[0]) & \
               (df_test["game_time_sec"] < phase_range[1])
        subset = df_test[mask]
        if len(subset) < 100:
            continue
        auc = roc_auc_score(subset["blue_win"], model.predict(subset[FEATURE_COLS]))
        print(f"{phase_name:12s}: AUC = {auc:.4f}  (n={len(subset):,})")
```

---

## 5. 모델 서빙 (추론 파이프라인)

```python
# scripts/inference/win_prob_predictor.py
"""
실시간 추론: Live Stats API 이벤트 → Win Probability 갱신
10초마다 호출 (매 초 갱신은 오버헤드 대비 정보 이득 낮음)
"""
import lightgbm as lgb
import numpy as np

class WinProbabilityPredictor:
    def __init__(self, model_path: str):
        self.model = lgb.Booster(model_file=model_path)
        self._history = []   # (timestamp, prob) 튜플 목록

    def predict(self, game_state: dict) -> float:
        """
        game_state dict → P(blue_win) ∈ [0, 1]

        game_state 예시:
        {
            "game_time_sec": 842,
            "gold_blue": 18500, "gold_red": 16200,
            "kills_blue": 7, "kills_red": 5,
            "towers_blue": 2, "towers_red": 1,
            "dragons_blue": 2, "dragons_red": 1,
            "barons_blue": 0, "barons_red": 0,
            "level_sum_blue": 47, "level_sum_red": 45,
            "cs_blue": 420, "cs_red": 380,
        }
        """
        t = game_state["game_time_sec"]
        phase = 0 if t < 900 else (1 if t < 1500 else 2)
        g_blue = game_state.get("gold_blue", 0)
        g_red = game_state.get("gold_red", 0)
        gold_diff = g_blue - g_red

        features = np.array([[
            t, phase,
            gold_diff,
            gold_diff / (t + 1),
            g_blue / max(g_blue + g_red, 1),
            game_state.get("kills_blue", 0) - game_state.get("kills_red", 0),
            game_state.get("towers_blue", 0) - game_state.get("towers_red", 0),
            game_state.get("dragons_blue", 0) - game_state.get("dragons_red", 0),
            game_state.get("barons_blue", 0) - game_state.get("barons_red", 0),
            game_state.get("level_sum_blue", 0) - game_state.get("level_sum_red", 0),
            game_state.get("cs_blue", 0) - game_state.get("cs_red", 0),
        ]])

        prob = float(self.model.predict(features)[0])
        self._history.append((t, prob))
        return prob

    def swing(self, window_sec: int = 30) -> float:
        """최근 window_sec 내 Win Probability 변화량 (한타 가치 지표)"""
        if len(self._history) < 2:
            return 0.0
        now_t = self._history[-1][0]
        recent = [(t, p) for t, p in self._history if now_t - t <= window_sec]
        if len(recent) < 2:
            return 0.0
        return abs(recent[-1][1] - recent[0][1])
```

---

## 6. 성능 목표 및 선행 연구 벤치마크

| 지표 | Gate A 목표 | 달성값 (EXP004) | 선행 연구 (arXiv 2023) |
|------|------------|--------------|----------------------|
| AUC-ROC | ≥ 0.83 (컨틴전시) | **0.8353** ✅ | 0.83 (30분 시점) |
| Brier Score | < 0.15 | 미측정 | ~0.13 |
| 추론 레이턴시 | < 5ms | < 5ms (로컬) | — |
| 학습 데이터 | 2,000경기 | 2,000경기 | ~50,000경기 |

**선행 연구 참고**: ["Real-time LoL Outcome Prediction" (arXiv:2309.02449)](https://arxiv.org/pdf/2309.02449)
- 10분 시점: AUC 0.63 → 20분: 0.75 → 30분: 0.83
- 핵심 피처: 골드차(1위), 타워차(2위), 킬차(3위) → EXP004 피처셋과 일치

---

## 7. 다음 단계

- [x] EXP001~006 실험 완료 → EXP004 채택 (AUC 0.8353)
- [x] GroupShuffleSplit 경기 단위 검증 적용
- [x] Calibration 측정 완료 — EXP005/006 ECE 0.1879→0.0269 (EXP004 AUC 우위로 EXP004 유지)
- [ ] Brier Score 공식 측정 (Platt Scaling/Isotonic Regression 적용 여부 판단 — Gate B)
- [ ] pgvectorscale 앙상블 통합 (empirical prob 베이즈 사전확률) — Gate C
- [ ] Bedrock 해설 연동: Win Probability swing 큰 한타 → 자동 하이라이트 트리거 — Gate B
- [ ] EXP-PG-001: Pre-game Win Pred (composition_embedding K-NN, AUC ≥ 0.70) — Gate B P2

## 8. 관련 파일

- [[data-pipeline-and-teamfight-detection]] — 학습 데이터 파이프라인
- [[champion-embedding-design]] — composition embedding 설계
- [[live-stats-api-analysis]] — 실시간 추론 입력 소스
- [[bedrock-prompt-templates]] — 해설 생성 연동
- [[gate-a-completion]] — Gate A 완료 보고 (EXP004 + EXP-TF-001 + Composition Embedding + Swing CSV)
- [[_index]] — 프로젝트 전체 개요
