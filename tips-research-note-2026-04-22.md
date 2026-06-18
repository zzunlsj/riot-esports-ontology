---
type: research
date: 2026-04-22
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [TIPS, 연구노트, ontology, LightGBM, TimescaleDB, Neo4j, teamfight, win-probability]
---

# TIPS 연구노트 — 이스포츠 실시간 경기 맥락 온톨로지 시스템 구현 현황

**과제명:** 리그 오브 레전드 실시간 경기 맥락 온톨로지 기반 AI 해설·예측 시스템  
**작성일:** 2026-04-22  
**작성자:** RORR (사업전략·상품전략·사업개발 팀장)  
**연구 단계:** 로컬 프로토타입 End-to-End 검증 완료 → 대규모 mock replay 확장 단계

---

## 1. 연구 배경 및 목표

본 과제는 리그 오브 레전드(LoL) 프로 경기 및 래더 경기 데이터를 실시간/준실시간으로 수집·분석하여, ① 한타 발생 사전 감지, ② 인게임 승률 예측(Win Probability), ③ 유저 수준별 AI 해설 자동 생성을 구현하는 온톨로지 기반 AI 서비스 개발을 목표로 한다.

| 목표 | 세부 내용 |
|------|---------|
| 1차 | 한타 발생 전 관전 시점·관전 포인트 자동 포착 |
| 2차 | 한타 종료 후 유저 수준별(L1~L4) 딥다이브 해설 생성 |
| 3차 | 사전·인게임 승부 예측(Pre-game / Win Probability / Teamfight Outcome) |

---

## 2. 시스템 아키텍처 개요

본 시스템은 정적 지식 그래프와 동적 시계열/벡터 레이어를 분리한 하이브리드 구조를 채택한다.

```
LAYER 3 : 추론·생성 레이어   — LightGBM + Bedrock(Claude API)
LAYER 2B: 벡터 레이어        — pgvectorscale (StreamingDiskANN)
LAYER 2A: 동적 온톨로지      — TimescaleDB (player_state / game_events / team_state)
LAYER 1 : 정적 온톨로지      — Neo4j (챔피언·아이템·팀·선수 그래프)
```

**핵심 설계 결정:**
- TimescaleDB와 pgvectorscale을 동일 PostgreSQL 인스턴스에서 운용하여 시계열 집계와 유사도 검색을 단일 SQL 계층에서 처리
- raw timeline/frame/event는 TimescaleDB에 저장하고, 패치/상성/시너지 같은 설명 가능한 관계는 Neo4j에 승격 저장
- mock replay를 통해 실시간 API 연결 전에도 `team_state` 기반 실시간 추론 검증을 가능하게 함

---

## 3. 2026-04-22 기준 최근 구현 업데이트

### 3-1. Match-v5 백필 파이프라인 고도화

`collect_matches.py`와 `backfill_matches.py`를 통해 단순 최근 경기 수집이 아니라 기간 기반 백필이 가능해졌다.

**핵심 구현:**
- `--start-date`, `--end-date`, `--pages`, `--page-size` 지원
- Riot Match-v5 `by-puuid/ids`의 `startTime` / `endTime` 필터 활용
- 기존 `*_timeline.json` 캐시를 읽어 중복 경기 자동 스킵
- 월 단위 배치 백필 오케스트레이터(`backfill_matches.py`) 추가

**의미:**
- “최근 경기 몇 개” 수준의 샘플링이 아니라, 작년~최근 구간을 학습 데이터셋으로 점진적으로 채우는 파이프라인이 생김
- 단, 현재 구조는 “현재 수집 대상 PUUID의 과거 경기”를 복원하는 방식이므로, 특정 시점 전체 래더 생태계를 완전 복원하는 것과는 다름

---

### 3-2. TimescaleDB 로컬 프로토타입 실제 운용 검증

이전에는 DDL 설계 문서 중심이었다면, 현재는 로컬 Docker 환경이 실제로 기동 중이며 스키마/확장/적재까지 확인된 상태다.

**확인 내용:**
- 컨테이너: `lol-ontology-db`
- 확장: `timescaledb`, `vector`, `vectorscale`
- 실제 사용 테이블:
  - `player_state`
  - `game_events`
  - `team_state`
  - `teamfight_events`
  - `composition_embeddings`

**현재 스키마 핵심:**
- 좌표계는 `pos_y`가 아니라 `pos_z`
- `team_state`에 실시간 예측 적재용 `win_prob`, `model_signal` 컬럼 포함

---

### 3-3. Win Probability 피처 재생성 및 모델 학습 완료

백필된 `player_state`, `game_events`를 바탕으로 `game_state_features.py`에서 학습용 parquet를 재생성하고, `train_win_prob.py`로 LightGBM 실험을 수행했다.

**최근 확인 수치:**

| 항목 | 값 |
|------|---|
| 피처 행 수 | 39,142 |
| 경기 수 | 1,497 |
| 레이블 평균 | 0.4889 |
| 시간 범위 | 0 ~ 45분 |

**실험 결과:**

| 실험 | AUC |
|------|-----|
| EXP001 | 0.8088 |
| EXP002 | 0.8088 |
| EXP003 | 0.8424 |

**현재 주력 모델:** `models/win_prob_EXP003.pkl`

**해석:**
- 골드 차이만으로도 0.80대 초반 AUC가 나오지만,
- `kill_diff`, `tower_diff`, `dragon_diff`, `team_level_diff`, `team_cs_diff`를 포함한 `EXP003`에서 목표치(0.80)를 넘는 성능을 확보
- 현 단계에서는 실시간 승률 추정의 POC 기준은 통과했다고 볼 수 있음

---

### 3-4. 실시간 추론 파이프라인을 `team_state`까지 연결

`live_stats_receiver.py`는 더 이상 단순 로그 수준이 아니라, 모델 추론 결과를 `team_state`에 시계열로 적재하는 단계까지 연결되어 있다.

**현재 동작:**
- 프레임마다 블루팀 기준 `win_prob` 계산
- `win_prob` 편차 기반 `model_signal` 생성
- `TeamfightDetector`의 룰 점수와 결합해 한타 alert 판단
- 팀 단위 상태를 `team_state`에 저장:
  - `total_gold`, `kills`, `towers`, `dragons`, `barons`, `inhibitors`, `cs`
  - `win_prob`, `model_signal`

**추가된 설명력:**
- `teamfight_detector.py`는 `rule_score`, `model_signal`을 함께 반환
- `live_stats_receiver.py` 로그에서 `win_prob`, `model_signal`, `rule_score`를 함께 출력
- 한타 종료 요약에는 `win_prob_swing`, `rule_score`, `model_signal_peak`가 포함

---

### 3-5. mock replay 기반 실시간 검증 확장

실시간 API 실경기 연결 전에도 캐시된 timeline을 재생하여 “실시간과 유사한” 추론 검증이 가능해졌다.

**검증 결과:**
- 단일 경기 `KR_7508335096`
  - `team_state` 적재 후 62행
  - 팀별 31프레임 저장 확인
- 3경기 replay
  - `team_state = 194행 / 3경기`

**전체 캐시 replay 진행 중간 수치:**

| 체크포인트 | team_state 행 수 | 경기 수 |
|-----------|------------------|--------|
| 시작 전 | 194 | 3 |
| 중간 1 | 760 | 12 |
| 중간 2 | 2,108 | 36 |
| 중간 3 | 3,488 | 63 |
| 중간 4 | 3,884 | 70 |
| 중간 5 | 5,516 | 103 |

즉, `replay_cached_matches.py --all`은 장시간 배치로 실제 `team_state`를 누적 확장하는 데 성공하고 있다.

---

### 3-6. 승률 급변 구간 분석 도구 추가

`win_prob_swings.py`를 추가해 `team_state.win_prob` 시계열에서 경기별 급변 프레임을 랭킹할 수 있게 했다.

**출력 컬럼 예시:**
- `match_id`
- `game_time_sec`
- `win_prob`
- `win_prob_prev`
- `swing_abs`
- `window_swing_abs`
- `model_signal`

**현재 상위 급변 샘플(중간 스냅샷):**
- `KR_7568622128` @ 1260초: `0.2487`
- `KR_7510276199` @ 2438초: `0.2046`
- `KR_7510276199` @ 2400초: `0.2021`

이 도구는 단순 모델 정확도 평가를 넘어, “어느 순간 게임 흐름이 실제로 가장 크게 뒤집혔는가”를 정량화하는 운영 분석 레이어 역할을 한다.

---

## 4. 현재 데이터 자산 현황

| 자산 | 규모 / 상태 |
|------|-------------|
| LCK 2025 시리즈 | 209개 |
| 게임 수 | 685개 |
| PlayerGameStats | 5,410개 |
| 챔피언 온톨로지 | 172개 챔피언, 280건 패치 변화 |
| WinProb 학습 피처 | 39,142행 / 1,497경기 |
| WinProb 최고 AUC | 0.8424 |
| 캐시 timeline | 1,950개 |
| team_state 전체 replay | 진행 중 |

---

## 5. 현재 기술적 의미

이번 단계에서 중요한 변화는 “설계”에서 “운영 가능한 로컬 프로토타입”으로 넘어갔다는 점이다.

1. **백필 → 학습 → 실시간 mock replay → 급변 분석**의 전체 루프가 실제 코드로 연결되었다.  
2. Win Probability 모델은 목표 AUC를 이미 넘었고, 단순 지표가 아니라 `team_state` 시계열에 적재되는 운영 산출물로 쓰이고 있다.  
3. 한타 감지 역시 룰 기반 감지기와 모델 시그널을 함께 노출하면서, 왜 alert가 발생했는지 해석 가능한 상태가 되었다.  
4. `win_prob_swings.py`를 통해 “이 모델이 어떤 경기 순간을 중요하게 봤는가”를 다경기 단위로 분석할 수 있게 되었다.  

즉, 현재 로컬 프로토타입은 단순히 모델을 학습한 수준이 아니라, “실시간 서비스의 축소판”으로 동작하고 있다.

---

## 6. 미완료 항목 및 향후 계획

| 항목 | 상태 | 비고 |
|------|------|------|
| Neo4j 운영 데이터 최신성 재검증 | 진행 필요 | 그래프 적재 최신 상태 확인 필요 |
| 챔피언 구성 임베딩 생성 | 미완료 | `champion_embeddings.py` 단계 |
| 전체 캐시 replay 완주 | 진행 중 | 완료 후 `team_state` 최종 총행 수 기록 필요 |
| 전수 swing 리포트 고정 | 미완료 | replay 완료 후 `win_prob_swings.py` 재실행 |
| Bedrock 실자격 증명 연동 | 미완료 | 현재는 mock commentary fallback |

---

## 7. 결론

2026-04-22 기준으로, Riot Esports 온톨로지 프로젝트는 다음 상태에 도달했다.

- TimescaleDB + pgvectorscale 로컬 프로토타입 실운용 검증 완료
- Match-v5 백필과 학습 파이프라인 검증 완료
- Win Probability 모델 목표 성능 확보 (`AUC 0.8424`)
- 실시간 mock replay 기반 `team_state` 시계열 적재 및 다경기 분석 검증 완료

남은 과제는 “기술 가능성 검증”이 아니라, **전체 replay 완주, 전수 리포트 생성, Neo4j/embedding 계층과의 통합 고도화**로 보는 것이 타당하다.

---

## 연결 노트

- [[progress-log]] — 최신 수치 / 진행 체크포인트
- [[implementation-plan-2026-04-11]] — 구현 플랜 및 완료 상태
- [[local-prototype-setup]] — 실제 로컬 스키마 / 운영 흐름
- [[win-probability-lightgbm]] — Win Probability 설계 / 실험 배경
- [[live-stats-api-receiver]] — 실시간 수신기 설계
- [[neptune-schema]] — 정적 온톨로지 스키마
