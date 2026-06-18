---
type: project
date: 2026-04-29
project: riot-esports-ontology
status: completed
tags: [gate-a, win-prediction, esports, lck]
---

# Gate A 완료 보고

> 완료일: 2026-04-29

## 결과 요약

| 실험 | 지표 | 결과 | Gate 기준 | 판정 |
|------|------|------|-----------|------|
| EXP004 Win Prob (LightGBM) | AUC | **0.8353** | ≥0.85 (컨틴전시 ≥0.83) | ✅ 통과 |
| EXP-TF-001 한타 결과 (XGBoost) | Accuracy | **83.4%** | ≥60% | ✅ 통과 |
| EXP-TF-001 한타 결과 (XGBoost) | AUC | **0.91** | — | ✅ |
| Composition Embedding (PCA 64) | 분산 설명률 | **91.3%** | 생성 여부 | ✅ 통과 |
| Win Prob Swing CSV | Top 50 생성 | **✅** | 생성 여부 | ✅ 통과 |

---

## EXP004 모델 상세

- **알고리즘**: LightGBM + GroupShuffleSplit (match-level CV)
- **학습 데이터**: 51,283 frames (2,000 경기 Match-v5 타임라인)
- **피처 16개**: baseline 9 + rich 7
  - baseline: gold_diff, kill_diff, tower_diff, dragon_diff, cs_diff, level_diff, gold_blue_pct, gold_diff_norm, game_time_min
  - rich: hp_diff, blue_hp_pct, red_hp_pct, dmg_share_diff, wards_diff, ward_kills_diff, kill_part_diff
- **모델 경로**: 

---

## EXP-TF-001 한타 결과 모델

- **알고리즘**: XGBoost (n_estimators=300, max_depth=5)
- **한타 수**: 8,711건 (2,000 타임라인에서 추출)
- **피처 중요도**: gold_diff(0.285) > level_diff(0.221) > hp_diff(0.152) > hp_pct × 2
- **모델 경로**: 

---

## Composition Embedding

- **데이터**: 786게임 × 2팀 = 1,572개 구성
- **챔피언 수**: 146개, one-hot → StandardScaler → PCA 64차원
- **Neo4j**:  노드 1,572개 적재
- **PCA 모델 경로**: 

---

## Win Prob Swing 대표 경기 5선

### Sample 1 — 후반 대역전극
- **match_id**: 
- **패턴**: 32분 한타에서 블루팀 우위(0.785) → 레드 역전(0.213), 42분 재역전(0.560)
- **스윙 크기**: 0.572 (전체 1위)
- **특이사항**: gold_diff 소폭(1,429)에도 kills 격차로 판세 전환

### Sample 2 — 중반 킬 사태 역전
- **match_id**: 
- **패턴**: 14분 블루 선제우위(0.719), 26분 추가 한타로 0.276 → 0.826 급상승
- **스윙 크기**: 0.549 (전체 2위)
- **특이사항**: kills=33, gold_diff=4,529 — 압도적 킬+골드 우위

### Sample 3 — 타워 싸움 롤러코스터
- **match_id**: 
- **패턴**: 21분 레드 역전(0.743→0.397), 23분 블루 반격(0.222→0.749)
- **스윙 크기**: 0.527
- **특이사항**: 2분 만에 판세 두 번 뒤집힘, 타워 5개 교환

### Sample 4 — 초반 판세 설정
- **match_id**: 
- **패턴**: 12분 블루 선제 주도권 상실(0.596→0.238), 21분 타워+킬로 복귀(0.667)
- **스윙 크기**: 0.358
- **특이사항**: 가장 이른 스윙(12분), 초반 draft/라인전 영향 포착

### Sample 5 — 장기전 오브젝트 압박
- **match_id**: 
- **패턴**: 24분 블루 급상승(0.735), 34분 재차 상승(0.833) — 타워 10개 파괴
- **스윙 크기**: 0.358
- **특이사항**: 34분까지 지속, max_towers=10 — 오브젝트 누적 승리 패턴

---

## 다음 단계 (Gate B)

- [ ] Bedrock 실제 자격증명 + E2E 파이프라인 연결
- [ ] WebSocket 레이턴시 측정 (목표: p95 < 200ms)
- [ ] Win Prob 실시간 오버레이 UI
- [ ] EXP-PG-001: Pre-game Win Pred K-NN (composition_embedding 활용)

---

## 연결 노트

- [[riot-esports-ontology]]
