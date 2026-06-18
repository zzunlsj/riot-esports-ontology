---
type: project
date: 2026-05-01
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [roadmap, OKR, metrics, kpi, milestone-gate, strategy]
---

# 목표지향 성공지표 및 플렉서블 로드맵

> **작성일**: 2026-04-29  
> **최종 업데이트**: 2026-05-01 — AUC 수치 정정 (0.8464 → 0.8353), team_state 실측값 반영  
> **기준 현황**: EXP004 AUC 0.8353 / EXP-TF-001 Acc 83.4% / Composition Embedding 1,572 노드 / Swing CSV 완료 → **Gate A 통과, Gate B 진입 준비**

---

## 1. 목표 체계

프로젝트의 3개 핵심 목표를 기반으로 성공지표와 로드맵을 연결한다.

| 목표 ID | 목표 | 서비스 페이즈 | 현재 단계 |
|--------|------|------------|---------|
| **G1** | 한타 발생 전 관전 시점·관전 포인트 자동 포착 | In-game | 규칙 엔진 설계 완료, ML 모델 미학습 |
| **G2** | 한타 종료 후 유저 수준별 딥다이브 해설 생성 | Post-fight | Bedrock 프롬프트 설계 완료, 실 자격증명 미연결 |
| **G3** | 사전·인게임 승부 예측 | Pre-game + In-game | Win Prob AUC **0.8353** (Gate A 통과), Teamfight Outcome Acc **83.4%**, Pre-game K-NN 미착수 |

---

## 2. 목표지향 성공지표 (KPI)

### 2-1. G1: 한타 포착 자동화

| 지표 | 정의 | 목표값 | 측정 방법 | 측정 시점 |
|-----|------|-------|---------|---------|
| Alert Precision | Alert 발동 건 중 실제 한타 발생 비율 | **≥ 75%** | teamfight_events vs Alert 로그 대조 | Gate C (내부 파일럿) |
| Alert Recall | 실제 한타 중 Alert 발동 비율 | **≥ 60%** | 동일 | Gate C |
| Alert 선행 시간 | 한타 시작 대비 Alert 발송 시점 (초 단위) | **≥ 15초** | Alert 타임스탬프 - fight 시작 타임스탬프 | Gate C |
| E2E 알림 레이턴시 | Alert 발동 → 클라이언트 수신까지 | **≤ 3초** | WebSocket 왕복 시간 측정 | Gate B |
| False Positive Rate | Alert 발동 건 중 한타 미발생 비율 | **≤ 25%** | = 1 - Precision | Gate C |

**Pivot 조건**: Gate C 기준 Precision < 50% → 임계값 재조정 (P > 0.75 → P > 0.65) 또는 LSTM 앙상블 가중치 재설계

---

### 2-2. G2: 맞춤형 해설 생성

| 지표 | 정의 | 목표값 | 측정 방법 | 측정 시점 |
|-----|------|-------|---------|---------|
| 해설 생성 레이턴시 | 한타 종료 감지 → L1 해설 완성까지 | **≤ 5초** | Bedrock invoke_model 응답 시간 | Gate B |
| 팩트 정확성 | 이벤트 수치(킬·골드·오브젝트) 기반 정확도 | **≥ 90%** | 해설 내 수치 vs teamfight_events 대조 | Gate C |
| 레벨 차별화 점수 | L1 vs L4 해설 간 내용 비중복률 | **≥ 60%** | ROUGE-L 역수 기반 차별화 측정 | Gate C |
| 월간 Bedrock 비용 | MVP 기준 월 Bedrock 호출 비용 | **≤ $500** | AWS 청구서 | Gate B 이후 |
| RAG 검색 정확도 | pgvectorscale 유사 한타 top-3 관련성 | **≥ 0.75 코사인** | 유사도 점수 분포 | Gate D |

**Pivot 조건**: Bedrock 비용 > $500 → L3/L4를 구독 Tier 전용으로 분리, L1/L2만 기본 제공하거나 응답 캐싱 레이어 추가

---

### 2-3. G3: 승부 예측 정확도

| 지표 | 정의 | 목표값 | 현재값 | 측정 방법 |
|-----|------|-------|------|---------|
| In-game Win Prob AUC | LightGBM 인게임 승률 예측 AUC | **≥ 0.85** (컨틴전시 ≥0.83) | **0.8353** ✅ (EXP004, 16피처) | GroupShuffleSplit 경기 단위 검증 |
| Pre-game Win Pred AUC | 챔피언 구성 기반 사전 승률 예측 | **≥ 0.70** | 미착수 (Gate B P2) | composition_embeddings K-NN 검증 |
| Teamfight Outcome Acc. | 한타 결과 이진 분류 정확도 | **≥ 65%** | **83.4%** ✅ (EXP-TF-001 XGBoost) | teamfight_embeddings hold-out 검증 |
| Win Prob 캘리브레이션 | 예측 확률 vs 실제 승률 RMSE | **≤ 0.10** | 미측정 | calibration curve 시각화 |
| Swing 탐지 정확도 | 결정적 한타(±15%p 이상) 포착률 | **≥ 80%** | Top 50 CSV 생성 완료 (수동 검증 미완) | win_prob_swings.py 결과 수동 검증 |

**Pivot 조건**: In-game AUC가 EXP005에서도 0.85 미달 시 → 쿨타임·포지션·와드 이벤트 피처 추가 실험. 여전히 미달 시 목표값 0.83으로 조정 후 Gate B 진행

---

## 3. 플렉서블 로드맵 (Gate 기반)

> **운영 원칙**: 각 Gate는 독립적인 진입 조건과 통과 기준을 갖는다. 기준 미달 시 해당 Gate 내 Contingency를 먼저 소진하고, 소진 후에도 미달이면 목표값 또는 범위를 조정한다. 병렬 진행 가능한 항목은 별도 표기한다.

```
[현재]
  ↓
Gate A: 오프라인 모델 성능 확정   (~2026-05-16)
  ↓
Gate B: 실시간 E2E 파이프라인 MVP (~2026-06-13)
  ↓
Gate C: 내부 파일럿 & 검증        (~2026-07-31)
  ↓
Gate D: 외부 파일럿 MVP           (~2026-10-31)
  ↓
Gate E: 플랫폼화 (수익화 진입)    (2027-Q2 이후)
```

---

### Gate A: 오프라인 모델 성능 확정 ✅ 완료
**완료일**: 2026-04-29

#### 진입 조건

- EXP003 AUC 0.8424 확인됨 (충족)
- TimescaleDB 로컬 컨테이너 동작 중 (충족)

#### 핵심 작업 (완료)

| 작업 | 결과 | 완료일 |
|-----|------|-------|
| 전체 캐시 replay 완료 → team_state 106,566행 / 2,000경기 | ✅ | 2026-04-29 |
| win_prob_swings.py --top 50 → Swing 랭킹 CSV 고정 | ✅ `data/win_prob_swings.csv` | 2026-04-29 |
| EXP004: HP/데미지/와드/킬참여율 피처 추가 (LightGBM 16피처) | ✅ AUC 0.8353 | 2026-04-29 |
| Composition Embedding(64차원) → Neo4j 1,572 노드 적재 | ✅ | 2026-04-29 |
| EXP-TF-001 Teamfight Outcome XGBoost 학습 | ✅ Acc 83.4% / AUC 0.91 | 2026-04-29 |
| Pre-game Win Pred K-NN (EXP-PG-001) | ⏩ Gate B P2로 이연 | — |

#### 통과 기준 (모두 충족 ✅)

- [x] In-game Win Prob AUC ≥ 0.83 (컨틴전시 기준 적용) — **달성: 0.8353**
- [x] Composition Embedding → Neo4j CompositionEmbedding 적재 완료 — **1,572 노드**
- [x] Teamfight Outcome Accuracy ≥ 60% — **달성: 83.4% / AUC 0.91**
- [x] Swing 랭킹 CSV 확정 + 대표 경기 샘플 5건 정리 — **완료**

#### 적용된 Contingency

- AUC 0.85 미달 → EXP005/006 재실험 (AUC 미개선, calibration만 개선 ECE 0.1879→0.0269) → **컨틴전시(≥0.83) 적용 후 통과**
- Composition Embedding: Phase 1 피처 엔지니어링(one-hot+PCA) 유지. DeepSets 고도화는 Gate D 이후로 이연

---

### Gate B: 실시간 E2E 파이프라인 MVP
**목표 완료**: 2026-06-13

#### 진입 조건

- Gate A 통과

#### 핵심 작업

| 작업 | 예상 기간 | 우선순위 | 병렬 여부 |
|-----|---------|---------|---------|
| Bedrock 실 자격증명 연결 + L1~L2 해설 실제 생성 테스트 | 1일 | P0 | — |
| Live Stats API 실제 경기 1게임 E2E 처리 (또는 고도화 mock) | 5일 | P0 | — |
| Alert → WebSocket → 클라이언트 수신 레이턴시 측정 | 2일 | P1 | 병렬 가능 |
| Win Probability 실시간 적재 → 웹 오버레이 시각화 초안 | 5일 | P1 | 병렬 가능 |
| Bedrock 월간 비용 추정 (LCK 1시즌 기준) | 1일 | P2 | 병렬 가능 |
| DynamoDB 현재 스냅샷 레이어 구현 | 5일 | P2 | 병렬 가능 |

#### 통과 기준

- [ ] E2E 해설 생성 레이턴시 ≤ 5초 (한타 감지 → L1 해설 클라이언트 수신)
- [ ] Alert 클라이언트 수신 레이턴시 ≤ 3초
- [ ] Bedrock 월 추정 비용 ≤ $500
- [ ] 1경기 완전 처리 로그 (한타 감지 → 해설 생성) 기록 완료

#### Contingency

- Live Stats API 실제 접근 불가 시: lolesports window endpoint 1초 폴링으로 대체, 실제 API 확보 후 교체
- Bedrock 비용 초과 시: L3/L4를 구독 Tier 전용으로 분리, L1/L2만 기본 제공

---

### Gate C: 내부 파일럿 & 검증
**목표 완료**: 2026-07-31

> **서비스 흐름 구분**: Gate C부터 두 흐름이 병렬 개발됩니다.
> - **Flow A** (실시간 해설): 이벤트 기반, Bedrock 단독
> - **Flow B** (쿼리 인터페이스): 사용자 질의, neo4j-graphrag-python + Bedrock

#### 진입 조건

- Gate B 통과
- LCK 실제 경기 대상 테스트 가능 환경 (또는 재생 mock 5경기 이상)

#### 핵심 작업

| 작업 | 흐름 | 예상 기간 | 우선순위 | 병렬 여부 |
|-----|------|---------|---------|---------|
| 실제 경기 5경기 이상 파이프라인 완전 처리 + 로그 기록 | A | 2주 | P0 | — |
| Alert Precision/Recall/선행시간 측정 및 분석 | A | 1주 | P0 | — |
| 해설 팩트 정확성 수동 검증 (경기당 3한타 샘플) | A | 1주 | P1 | 병렬 가능 |
| 웹 오버레이 MVP 완성 (Win Prob 바 + 한타 해설 패널) | A | 3주 | P1 | 병렬 가능 |
| LSTM 한타 예측 모델 학습 (rule-based 대비 Precision 비교) | A | 3주 | P1 | 병렬 가능 |
| Pre-game 승률 예측 UI 연결 | A | 2주 | P2 | 병렬 가능 |
| MiroFish 선수 행동 속성 추출 파이프라인 | A | 2주 | P1 | 병렬 가능 |
| **neo4j-graphrag-python Text2Cypher 구현** | **B** | **2일** | **P1** | 병렬 가능 |
| **도메인 예시 쌍 작성 (질문 → Cypher 10개)** | **B** | **1일** | **P1** | 병렬 가능 |
| **애널리스트 쿼리 인터페이스 내부 검증** | **B** | **1주** | **P2** | 병렬 가능 |
| 내부 검토자 피드백 수집 (3인 이상) | A+B | 1주 | P2 | — |

#### 통과 기준

- [ ] Alert Precision ≥ 65% (외부 파일럿 전 완화 기준, Gate D에서 75%로 상향)
- [ ] Alert Recall ≥ 55%
- [ ] 해설 팩트 정확성 ≥ 90%
- [ ] 웹 오버레이 MVP 데모 가능 상태
- [ ] 내부 피드백 긍정 평가 ≥ 70%
- [ ] **Text2Cypher 쿼리 정확도 ≥ 80%** (내부 검증 질문 10개 기준)

#### Contingency

- Precision 미달 시: 임계값 P > 0.75 → P > 0.70으로 조정, 또는 LSTM 앙상블 도입 시점 앞당김
- LSTM이 rule-based 대비 개선 없을 시: 앙상블 가중치 재조정 (LSTM 0.5 → 0.3), 추가 학습 데이터 확보 검토
- Text2Cypher 정확도 미달 시: 예시 쌍 추가 (10 → 20개), 스키마 설명 보강

---

### Gate D: 외부 파일럿 MVP
**목표 완료**: 2026-10-31

#### 진입 조건

- Gate C 통과
- 외부 파일럿 대상 확보 (이스포츠 팬 커뮤니티 또는 방송 협력사)

#### 핵심 작업

| 작업 | 흐름 | 예상 기간 | 우선순위 | 병렬 여부 |
|-----|------|---------|---------|---------|
| LSTM 한타 예측 고도화 (Precision ≥ 75% 목표) | A | 4주 | P0 | — |
| EC2 이전 (Neo4j AuraDB + TimescaleDB) | A+B | 2주 | P0 | 병렬 가능 |
| L3/L4 해설 생성 + pgvectorscale RAG 연결 | A | 4주 | P1 | 병렬 가능 |
| 모바일 웹 오버레이 최적화 | A | 4주 | P1 | 병렬 가능 |
| 경기 결정 한타 TOP 3 리포트 기능 구현 | A | 2주 | P2 | 병렬 가능 |
| **애널리스트·코치 쿼리 인터페이스 외부 공개** | **B** | **2주** | **P1** | 병렬 가능 |
| **MiroFish 선수 에이전트 쿼리 예시 확장** | **B** | **1주** | **P2** | 병렬 가능 |
| **가상 로스터 구성 + Monte Carlo 시뮬레이터 MVP** | **B** | **4주** | **P2** | 병렬 가능 |
| 수익화 모델 A/B 테스트 설계 | A+B | 1주 | P3 | — |

#### 통과 기준

- [ ] Alert Precision ≥ 75%
- [ ] Alert Recall ≥ 60%
- [ ] L1~L4 해설 모두 실제 경기 대상 동작
- [ ] 외부 파일럿 사용자 만족도 ≥ 70% (5점 척도 기준)
- [ ] 클라우드 E2E 레이턴시 ≤ 5초 (EC2 기준)
- [ ] **애널리스트 쿼리 인터페이스 외부 파일럿 3인 이상 검증 완료**

#### Contingency

- 외부 파일럿 협력사 확보 불가 시: 유튜브 VOD 기반 재생 데모로 대체, 사용자 풀 확보 후 재진행
- 클라우드 비용 초과 시: Timescale Cloud 대신 EC2 단일 인스턴스 유지 (상용화 시점에 재검토)

---

### Gate E: 플랫폼화 (수익화 진입)
**목표 완료**: 2027-Q2 이후 (Gate D 결과에 따라 재설정)

#### 조건부 진입

- Gate D 통과 AND (유료 전환율 ≥ 5% 또는 B2B 협력사 LOI 1곳 이상 확보)

#### 주요 방향

| 방향 | 설명 | 단계 |
|-----|------|------|
| Freemium | L1~L2 무료 / L3~L4 구독 ($9.9/월) | MVP |
| B2B 방송사 | OBS 플러그인 + 데이터 피드 공급 | 2단계 |
| 팀/코치 라이선스 | Analyst 기능 + API 접근 ($299/월) | 2단계 |
| AWS Kinesis 도입 | 동시 경기 ≥ 5개 운영 시 팬아웃 파이프라인 전환 | 수요 확인 후 |
| Neo4j AuraDB 이전 | Neo4j Desktop → Neo4j AuraDB (관리형 클라우드, Bolt 호환) | Gate D 클라우드 이전 시 |

---

## 4. 현재 위치 진단 (2026-04-29 기준)

| 항목 | 상태 | 비고 |
|-----|------|------|
| **Gate A** | ✅ **완료** | 2026-04-29 |
| In-game Win Prob AUC | ✅ **0.8353** (EXP004, LightGBM 16피처) | 컨틴전시 ≥0.83 적용 통과 |
| Composition Embedding | ✅ **완료** | Neo4j 1,572 CompositionEmbedding 노드 / PCA 64차원 91.3% 분산 |
| Teamfight Outcome 모델 | ✅ **완료** | Acc 83.4% / AUC 0.91 (EXP-TF-001 XGBoost) |
| 전체 캐시 replay | ✅ **완료** | team_state 106,566행 / 2,000경기 |
| Swing 랭킹 CSV | ✅ **완료** | Top 50 + 대표 5경기 선정 |
| **Gate B** | 🔜 **진입 준비 중** | — |
| Bedrock 실 자격증명 | ❌ 미연결 | Gate B P0 |
| LoL Esports 실 경기 E2E | ❌ mock 단계 | Gate B P0 (lolesports_live_poller.py 준비 완료) |
| WebSocket 레이턴시 측정 | ❌ 미착수 | Gate B P1 |
| Win Prob 실시간 오버레이 UI | ❌ 미착수 | Gate B P1 |
| EXP-PG-001 Pre-game K-NN | ❌ 이연 | Gate B P2 |
| 웹 오버레이 | ❌ 미착수 | Gate C |

---

## 5. 즉시 액션 아이템 (Gate B 진입, 2026-04-29 ~ 2026-05-31)

| 우선순위 | 액션 | 산출물 |
|---------|------|------|
| P0 | Bedrock 실 자격증명 연결 + L1~L2 해설 실제 생성 테스트 | 해설 생성 레이턴시 측정값 |
| P0 | lolesports_live_poller로 과거 경기 replay → E2E 완전 처리 | 1경기 완전 처리 로그 |
| P1 | WebSocket Alert → 클라이언트 수신 레이턴시 측정 | p95 레이턴시 수치 |
| P1 | Win Probability 실시간 적재 → 웹 오버레이 시각화 초안 | 오버레이 스크린샷 |
| P2 | EXP-PG-001: composition_embedding 기반 K-NN Pre-game 예측 | Pre-game AUC |

---

## 6. 변경 이력

| 날짜 | 변경 내용 | 사유 |
|-----|---------|------|
| 2026-04-29 | 최초 작성 | 프로젝트 현황 기반 목표지향 지표 및 Gate 로드맵 수립 |
| 2026-04-29 | Gate A 완료 처리 + 현황 진단 업데이트 | EXP004/EXP-TF-001/Composition Embedding/Swing CSV 전 항목 완료 |
| 2026-05-01 | AUC 수치 정정 + team_state 실측값 반영 | 0.8464 → 0.8353 (전체), 51,478행/969경기 → 106,566행/2,000경기 |

---

## 연결 문서

- [[_index]] — 기술 아키텍처 전체
- [[service-scenario]] — 서비스 시나리오 및 유저 여정
- [[progress-log]] — 진행 상태 상세 로그
- [[win-probability-lightgbm]] — Win Prob 실험 상세
- [[implementation-plan-2026-04-11]] — 기술 구현 플랜
