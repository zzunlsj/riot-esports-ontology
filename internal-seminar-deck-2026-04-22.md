---
type: presentation-draft
date: 2026-04-22
project: Riot Esports 실시간 경기 맥락 온톨로지
status: completed
tags: [seminar, gamma, notebooklm, slides, internal]
---

# Riot Esports 실시간 경기 맥락 온톨로지
## 내부 세미나 자료 초안

부제: 로컬 프로토타입 End-to-End 검증 현황과 다음 단계

작성일: 2026-04-22

---

# 1. 왜 이 프로젝트를 하는가

- LoL 경기 시청 경험은 여전히 "무슨 일이 왜 중요한가"를 실시간으로 이해하기 어렵다
- 특히 한타 직전 맥락, 승률 변화, 오브젝트 전환의 의미를 자동으로 설명하는 계층이 부족하다
- 목표는 경기 데이터에서 맥락을 추출해:
  - 한타 발생 전 관전 포인트를 포착하고
  - 경기 중 승률을 추정하고
  - 한타 종료 후 유저 수준별 해설을 생성하는 것이다

핵심 질문:
- 지금 벌어지는 싸움은 왜 중요한가?
- 어느 팀이 유리한가?
- 이 장면을 초보자와 숙련자에게 각각 어떻게 설명할 것인가?

---

# 2. 우리가 만들고 있는 것

- "정적 지식 그래프 + 동적 시계열 + 벡터 검색 + 생성형 해설"을 결합한 이스포츠 온톨로지 시스템
- 단순 통계 대시보드가 아니라:
  - 설명 가능한 관계 지식
  - 실시간 상태 추론
  - 유사 과거 상황 검색
  - 수준별 해설 생성
  를 하나의 파이프라인으로 연결한다

최종 서비스 상:
- 한타 직전 알림
- 인게임 승률 곡선
- 한타 후 자동 해설
- 향후에는 프리게임 밴픽/조합 기반 프리뷰까지 확장 가능

---

# 3. 시스템 아키텍처 한 장 요약

LAYER 1: Neo4j
- 챔피언, 아이템, 팀, 선수, 상성, 시너지 같은 정적 지식 그래프

LAYER 2A: TimescaleDB
- `player_state`, `game_events`, `team_state`, `teamfight_events`
- 학습용 백필과 실시간 상태 저장

LAYER 2B: pgvectorscale
- 상태 임베딩, 조합 임베딩, 유사 상황 검색

LAYER 3: 추론 및 생성
- LightGBM 기반 Win Probability
- 룰 기반 + 모델 시그널 기반 한타 감지
- Bedrock/Claude 기반 해설 생성

핵심 설계 결정:
- TimescaleDB와 pgvectorscale을 같은 PostgreSQL 인스턴스에 두어 시계열과 유사도 검색을 단일 계층에서 처리

---

# 4. 현재 구현 범위

- Neo4j 스키마와 파이프라인 코드 정리 완료
- TimescaleDB 로컬 Docker 프로토타입 구축 및 확장 확인 완료
- Match-v5 timeline 백필 파이프라인 구현 완료
- Win Probability 피처 생성 및 LightGBM 학습 완료
- 실시간 mock replay 기반 추론 검증 완료
- `team_state.win_prob`, `team_state.model_signal` 적재 확인 완료
- 승률 급변 구간 분석 도구 추가 완료

요약:
- 설계 문서 단계가 아니라 "작동하는 로컬 프로토타입" 단계에 도달

---

# 5. 데이터 파이프라인

수집 경로 1: 정적/준정적 데이터
- LCK 경기 메타데이터
- 선수/팀/로스터
- 챔피언 온톨로지
- 패치 변화

수집 경로 2: 동적 timeline 데이터
- Riot Match-v5 timeline
- `collect_matches.py`
- `backfill_matches.py`

현재 백필 방식:
- 현재 대상 PUUID의 과거 경기들을 기간 기준으로 수집
- `start-date`, `end-date`, `pages`, `page-size` 지원
- 캐시된 `*_timeline.json` 기반 중복 스킵

---

# 6. TimescaleDB에 무엇이 쌓이는가

`player_state`
- 참가자 단위 상태
- 위치, 생존 여부, 체력/자원, 레벨, 골드, CS, 쿨타임, 상태 임베딩

`game_events`
- 킬, 건물 파괴, 엘리트 몬스터, 와드, 아이템 이벤트

`team_state`
- 팀 단위 집계
- 골드, 킬, 타워, 드래곤, 바론, 억제기, CS
- 추가로 실시간 추론 결과:
  - `win_prob`
  - `model_signal`

`teamfight_events`
- 감지된 한타 요약
- 지속 시간, 골드 스윙, 킬 수, 오브젝트 전환 등

---

# 7. 지금 확보한 학습 자산

최근 확인 기준:
- Win Probability 학습 피처: 39,142행
- 경기 수: 1,497경기
- 레이블 평균: 0.4889
- 시간 범위: 0분 ~ 45분

별도 자산:
- LCK 2025 시리즈: 209개
- 게임 수: 685개
- PlayerGameStats: 5,410개
- 챔피언 온톨로지: 172개 챔피언, 280건 패치 변화
- 캐시 timeline 파일: 1,950개

의미:
- 더 이상 toy dataset이 아니라 실험 가능한 규모의 경기 상태 데이터셋 확보

---

# 8. Win Probability 모델 결과

실험 결과:
- EXP001: 0.8088
- EXP002: 0.8088
- EXP003: 0.8424

현재 주력 모델:
- `win_prob_EXP003.pkl`

중요 포인트:
- 골드 기반 베이스라인만으로도 0.80 이상
- 킬, 타워, 드래곤, 팀 레벨, 팀 CS 차이를 포함한 EXP003에서 목표치를 넘김
- 현재 단계에서 "실시간 승률 추정" POC는 통과로 볼 수 있음

---

# 9. 한타 감지 로직

Rule 1
- 적 2인 이상이 1000 유닛 이내 집결
- 아군 1인 이상 근접

Rule 2
- 오브젝트 근처에 양팀 모두 존재

Rule 3
- 최근 10초 내 와드 2개 이상 파괴

추가 신호:
- `win_prob` 편차 기반 `model_signal`

현재 구조:
- 룰 점수와 모델 시그널을 함께 사용
- 로그에 `rule_score`, `model_signal`을 함께 출력
- alert가 왜 발생했는지 해석 가능

---

# 10. 실시간 mock replay 검증

왜 mock replay가 중요한가:
- 실제 Live Stats 실경기 연결 전에도 파이프라인을 end-to-end로 검증 가능

검증 결과:
- 단일 경기 `KR_7508335096`
  - `team_state` 62행 적재
  - 팀별 31프레임 저장 확인

- 3경기 replay
  - `team_state = 194행 / 3경기`

- 전체 캐시 replay 진행 중간 체크
  - 760행 / 12경기
  - 2108행 / 36경기
  - 3488행 / 63경기
  - 3884행 / 70경기
  - 5516행 / 103경기

핵심 의미:
- 실시간 추론 시계열이 실제 DB에 누적 저장되고 있음

---

# 11. 승률 급변 구간 분석

새 도구:
- `win_prob_swings.py`

역할:
- `team_state.win_prob` 시계열에서 급변 프레임을 랭킹
- 경기 전반에서 "가장 의미 있는 전환점"을 정량화

출력 컬럼 예시:
- `match_id`
- `game_time_sec`
- `win_prob`
- `win_prob_prev`
- `swing_abs`
- `window_swing_abs`
- `model_signal`

활용 가능성:
- 경기 리뷰
- 하이라이트 후보 추출
- 한타 요약 우선순위화

---

# 12. 현재까지의 의미

이번 단계의 본질은:
- 개념 설계에서
- 작동하는 로컬 프로토타입으로 넘어갔다는 점이다

이제 가능한 것:
- 과거 데이터 백필
- 피처 생성과 모델 학습
- 실시간처럼 경기 replay
- 실시간 승률 곡선 적재
- 경기 간 급변 시점 랭킹

즉, "모델 한 개를 학습했다"가 아니라
"서비스 축소판을 한 번 끝까지 돌릴 수 있다"가 현재 성과다

---

# 13. 아직 안 된 것

- Neo4j 운영 데이터 최신성 재검증
- 챔피언 조합 임베딩 생성
- pgvectorscale 기반 empirical prior 본격 결합
- 전체 1,950경기 replay 완주 및 전수 리포트 고정
- Bedrock 실자격 증명 연결 테스트
- 실경기 Live Stats 연동 검증

현재 단계의 한계:
- mock replay는 "실시간과 유사한 검증"이지 실경기 스트림 검증 자체는 아님
- 현재 백필은 "현재 수집 대상 PUUID의 과거 경기" 복원 방식

---

# 14. 다음 단계 제안

1. 전체 캐시 replay 완주
- `team_state` 최종 총행 수/경기 수 확정
- 전수 swing 리포트 재생성

2. 대표 급변 경기 샘플 정리
- 팀 내부 데모용 사례 선정
- 해설 품질 검토

3. composition embedding 단계 진입
- 조합 기반 prior와 outcome 예측 확장

4. Neo4j 관계 승격 자동화
- COUNTERS / SYNERGIZES_WITH / patch-context 관계 생성

5. 실경기 Live Stats 연결 테스트
- mock에서 운영 검증 단계로 이동

---

# 15. 팀에 공유하고 싶은 핵심 메시지

- 이 프로젝트는 아직 "아이디어 검토 단계"가 아니다
- 이미 작동하는 로컬 프로토타입이 있고, 주요 리스크는 기술 가능성이 아니라 운영 고도화 쪽으로 이동했다
- 특히 TimescaleDB 중심의 동적 온톨로지 레이어가 실제로 학습과 실시간 검증 둘 다를 받쳐주고 있다
- 다음 분기 논의는 "할 수 있나?"보다 "어디에 우선 적용하고 어떻게 보여줄까?"에 가까워져야 한다

---

# 16. 부록: 재현용 운영 루프

```bash
cd 1-projects/riot-esports-ontology/scraper
source .venv/bin/activate

# 1. 백필
python scripts/pipeline/backfill_matches.py \
  --start-month 2025-01 \
  --end-month 2026-04 \
  --target-per-month 100 \
  --pages 5 \
  --page-size 50

# 2. 피처 / 학습
python scripts/features/game_state_features.py
python scripts/models/train_win_prob.py

# 3. mock replay
python -m scripts.realtime.replay_cached_matches --limit 20

# 4. 승률 급변 분석
python scripts/features/win_prob_swings.py --top 50 --window-size 1
```

---

# 17. 참고 문서

- `progress-log.md`
- `implementation-plan-2026-04-11.md`
- `local-prototype-setup.md`
- `win-probability-lightgbm.md`
- `live-stats-api-receiver.md`
- `neptune-schema.md`

---

# 18. 세미나 이후 업데이트 (Gate A 완료 · 2026-04-29)

> 이 슬라이드는 2026-04-22 발표 이후 달라진 수치를 기록합니다. 슬라이드 §7~§14는 당시 상태 그대로 보존.

| 항목 | 발표 당시 (2026-04-22) | Gate A 완료 (2026-04-29) |
|-----|---------------------|------------------------|
| 캐시 timeline 파일 | 1,950개 (진행 중) | **2,000경기** (완주) |
| Win Prob 학습 피처 | 39,142행 / 1,497경기 | **51,283행 / 2,000경기** |
| team_state 적재 | 진행 중 (부분) | **106,566행 / 2,000경기** |
| teamfight_events | — | **8,711건** |
| 최고 모델 | EXP003 AUC 0.8424 | **EXP004 AUC 0.8353** (LightGBM 16피처) |
| Teamfight Outcome | — | **EXP-TF-001 Acc 83.4% / AUC 0.91** (XGBoost) |
| 조합 임베딩 | 미착수 | **1,572 nodes / PCA 64dim** (Neo4j 적재 완료) |
| Swing CSV | — | **Top 50 고정** (`win_prob_swings.csv`) |
| Bedrock 자격증명 | 미착수 | ❌ 미연결 (mock 모드, Gate B P0) |

**§13 "아직 안 된 것" 중 완료된 항목**:
- ✅ 챔피언 조합 임베딩 생성 완료
- ✅ 전체 캐시 replay 완주 및 전수 리포트 고정
- ❌ Bedrock 실자격 증명 연결 → Gate B 이월
- ❌ 실경기 Live Stats 연동 검증 → Gate B 이월

**§14 "다음 단계" 중 완료된 항목**:
- ✅ 전체 캐시 replay 완주 + swing 리포트 재생성
- ✅ 대표 급변 경기 샘플 5건 정리
- ✅ Composition Embedding 단계 진입 및 완료
- ⬜ Neo4j 관계 승격 자동화 (Gate B)
- ⬜ 실경기 Live Stats 연결 테스트 (Gate B)
