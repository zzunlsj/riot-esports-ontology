---
type: research
date: 2026-04-30
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [embedding, champion, ML, vector, design]
---

# 챔피언 구성 임베딩 모델 설계

> 5챔피언 → 64차원 구성 벡터(Composition Vector) 설계.
> 목적: Pre-game 승률 예측 + 유사 구성 K-NN 검색.

---

## 0. 현재 구현 상태 (Gate A · 2026-04-30)

| 컴포넌트 | 상태 | 비고 |
|---------|------|------|
| Phase 1 피처 기반 임베딩 (20dim × 5 → 64dim) | ✅ 설계 완료 | §1 코드 참조 |
| composition_embeddings 테이블 적재 | ✅ 완료 | **1,572 nodes, PCA 64dim**, TimescaleDB |
| K-NN 유사 구성 검색 (pgvectorscale) | ⚫ 미구현 | Gate B 예정 |
| Phase 2 DeepSets 학습 기반 임베딩 | ⚫ 미구현 | Gate C 이후 |
| Pre-game 승률 예측 연동 (EXP-PG-001) | ⚫ 미착수 | Gate B 예정 |

> **Architecture 표기**: `composition_embeddings 1,572 nodes (PCA 64dim)` — Phase 1 feature vector 생성 후 PCA로 압축한 결과물.
> K-NN 인덱스(diskann)는 아직 미빌드.

---

## 설계 원칙

1. **순서 불변(permutation-invariant)**: 픽 순서가 달라도 같은 팀 구성이면 동일한 벡터
2. **의미 보존**: 비슷한 역할/특성의 구성은 벡터 공간에서 가까워야 함
3. **경량성**: 추론 레이턴시 < 5ms, 학습 없이 업데이트 가능 (패치 대응)
4. **해석 가능성**: 각 차원이 게임 속성에 매핑되어야 디버깅 가능

---

## 접근법 비교

| 방식 | 장점 | 단점 | 선택 |
|------|------|------|------|
| One-hot (165차원 × 5) | 단순 | 순서 의존, 극고차원 | ❌ |
| Sum of champion embeddings | 순서 불변, 학습 불필요 | 개별 임베딩 품질 의존 | ✅ Phase 1 |
| DeepSets (학습 기반) | 최고 성능 | 학습 데이터·시간 필요 | ✅ Phase 2 |
| Autoencoder | 비지도 학습 | 해석 어려움 | 보류 |

**단계별 전략**: Phase 1은 피처 엔지니어링 기반 수작업 임베딩 → Phase 2에서 DeepSets로 업그레이드

---

## Phase 1: 피처 기반 구성 임베딩 (64차원)

### 개별 챔피언 피처 벡터 (20차원)

```python
def champion_feature_vector(champion_id: str, patch_data: dict) -> np.ndarray:
    """
    챔피언 1개 → 20차원 피처 벡터
    모든 값은 0~1 정규화
    """
    champ = patch_data[champion_id]

    vec = np.array([
        # [0] 역할 원-핫 (5차원): top, jungle, mid, bot, support
        int('top' in champ['roles']),
        int('jungle' in champ['roles']),
        int('mid' in champ['roles']),
        int('bot' in champ['roles']),
        int('support' in champ['roles']),

        # [5] 클래스 원-핫 (6차원)
        int(champ['class'] == 'assassin'),
        int(champ['class'] == 'fighter'),
        int(champ['class'] == 'mage'),
        int(champ['class'] == 'marksman'),
        int(champ['class'] == 'support'),
        int(champ['class'] == 'tank'),

        # [11] 전투 속성 (5차원) — 0~1 정규화
        champ['engage_potential'],        # 0(없음)~1(최상) ex. Leona=1.0, Jinx=0.0
        champ['cc_count'] / 5.0,          # CC 스킬 수 / 5
        champ['burst_damage'],            # 0~1
        champ['sustained_damage'],        # 0~1
        champ['mobility'],                # 이동기 보유 점수

        # [16] 사거리 (1차원)
        min(champ['attack_range'] / 700.0, 1.0),  # 최대 700 유닛으로 정규화

        # [17] 성장형 (1차원) — 후반 강화 지수
        champ['late_game_scaling'],       # 0~1

        # [18] 초반형 (1차원)
        champ['early_game_power'],        # 0~1

        # [19] 스케일링 타입 인코딩 (1차원)
        {'AD': 0.0, 'AP': 0.5, 'hybrid': 0.75, 'tank': 1.0}[champ['scaling_type']]
    ], dtype=np.float32)

    return vec  # shape: (20,)
```

### 팀 구성 집계 (20차원 × 5 → 64차원)

```python
def composition_embedding(
    champion_ids: list[str],  # 5개 챔피언 ID 리스트
    patch_data: dict
) -> np.ndarray:
    """
    5챔피언 → 64차원 composition vector
    순서 불변(permutation-invariant) 보장
    """
    assert len(champion_ids) == 5

    # 각 챔피언 피처 계산
    vecs = np.array([champion_feature_vector(cid, patch_data) for cid in champion_ids])
    # shape: (5, 20)

    # ── 집계 레이어 (순서 불변) ──────────────────────────────

    # 1. 합계 (20차원): 팀 전체 특성 강도
    sum_vec = vecs.sum(axis=0)           # (20,)

    # 2. 최댓값 (20차원): 팀에서 가장 높은 속성
    max_vec = vecs.max(axis=0)           # (20,)

    # 3. 특수 구성 지표 (24차원): 게임 전략 시그니처
    meta_vec = np.array([
        # 역할 커버리지: 5개 역할 모두 커버하는가?
        float(sum_vec[0] > 0),   # top 있음
        float(sum_vec[1] > 0),   # jungle 있음
        float(sum_vec[2] > 0),   # mid 있음
        float(sum_vec[3] > 0),   # bot 있음
        float(sum_vec[4] > 0),   # support 있음

        # 이니시에이터 수 (engage 챔피언 몇 명?)
        np.sum(vecs[:, 11] > 0.7) / 5.0,  # high engage 비율

        # CC 집약도
        sum_vec[12] / 5.0,               # 팀 평균 CC

        # 딜러 비율 (assassin + marksman + mage)
        np.sum(vecs[:, 5] + vecs[:, 7] + vecs[:, 8]) / 5.0,

        # 탱커 비율 (tank + fighter)
        np.sum(vecs[:, 6] + vecs[:, 9]) / 5.0,

        # 평균 사거리 (원딜 조합 여부)
        sum_vec[16] / 5.0,

        # 포킹 잠재력 (긴 사거리 + 지속 딜 챔피언 수)
        np.sum((vecs[:, 16] > 0.6) & (vecs[:, 14] > 0.5)) / 5.0,

        # 스플릿 잠재력 (이동기 + 지속 딜)
        np.sum((vecs[:, 15] > 0.6) & (vecs[:, 14] > 0.5)) / 5.0,

        # 후반 조합 강도
        sum_vec[17] / 5.0,

        # 초반 조합 강도
        sum_vec[18] / 5.0,

        # 버스트 조합 강도 (암살자/원거리 마법사 버스트)
        sum_vec[13] / 5.0,

        # 지속전 강도
        sum_vec[14] / 5.0,

        # AP 챔피언 수
        np.sum(vecs[:, 19] >= 0.5) / 5.0,

        # AD 챔피언 수
        np.sum(vecs[:, 19] == 0.0) / 5.0,

        # 챔피언 다양성 (클래스 엔트로피)
        _class_entropy(vecs) / np.log(6),  # 0~1 정규화

        # 이동기 집약도
        sum_vec[15] / 5.0,

        # 기타 패딩 (향후 확장용)
        0.0, 0.0, 0.0, 0.0
    ], dtype=np.float32)
    # shape: (24,)

    # 최종 결합: 20 + 20 + 24 = 64차원
    comp_vec = np.concatenate([sum_vec, max_vec, meta_vec])
    assert comp_vec.shape == (64,)

    return comp_vec


def _class_entropy(vecs: np.ndarray) -> float:
    """팀 구성의 클래스 다양성 (Shannon entropy)"""
    class_counts = vecs[:, 5:11].sum(axis=0)  # (6,) 각 클래스 수
    total = class_counts.sum()
    if total == 0:
        return 0.0
    probs = class_counts / total
    probs = probs[probs > 0]
    return float(-np.sum(probs * np.log(probs)))
```

---

## Phase 2: DeepSets 학습 기반 임베딩 (향후)

```python
import torch
import torch.nn as nn

class ChampionEncoder(nn.Module):
    """단일 챔피언 → 32차원 잠재 벡터"""
    def __init__(self, num_champions=170, embed_dim=32):
        super().__init__()
        self.embed = nn.Embedding(num_champions, embed_dim)
        self.fc = nn.Sequential(
            nn.Linear(embed_dim, 64), nn.ReLU(),
            nn.Linear(64, 32)
        )

    def forward(self, champ_id):  # (batch,)
        return self.fc(self.embed(champ_id))  # (batch, 32)


class DeepSetsCompositionEncoder(nn.Module):
    """
    5챔피언 → 64차원 구성 벡터
    DeepSets: permutation-invariant by design
    """
    def __init__(self):
        super().__init__()
        self.phi = ChampionEncoder(embed_dim=32)  # 개별 인코더
        self.rho = nn.Sequential(                  # 집계 후 변환
            nn.Linear(32, 64), nn.ReLU(),
            nn.Linear(64, 64)
        )

    def forward(self, champ_ids):  # (batch, 5)
        # 개별 임베딩
        encoded = torch.stack(
            [self.phi(champ_ids[:, i]) for i in range(5)], dim=1
        )  # (batch, 5, 32)

        # 합산 집계 (순서 불변)
        aggregated = encoded.sum(dim=1)  # (batch, 32)

        # 최종 변환
        return self.rho(aggregated)  # (batch, 64)


# 학습 목표: 같은 팀 구성으로 이긴 경기 쌍은 가깝게, 진 경기 쌍은 멀게
# Loss: Triplet Loss 또는 Contrastive Loss (win/lose label)
```

---

## 임베딩 품질 평가 지표

> **Gate A 상태**: K-NN 인덱스 미빌드 → 검증 2 실행 불가. Gate B에서 측정 예정.

```python
# 검증 1: 유사 조합 근접성
# "탱커+이니시" 조합끼리 유사도가 높아야 함
leona_alistar_comp = composition_embedding(['Leona', 'Alistar', 'Garen', 'Jinx', 'Orianna'], ...)
leona_nautilus_comp = composition_embedding(['Leona', 'Nautilus', 'Darius', 'Jhin', 'Syndra'], ...)
poke_comp = composition_embedding(['Jayce', 'Nidalee', 'Zoe', 'Ezreal', 'Karma'], ...)

# 기대: leona_alistar vs leona_nautilus 유사도 > leona_alistar vs poke
from numpy.linalg import norm
def cosine_sim(a, b):
    return np.dot(a, b) / (norm(a) * norm(b))

assert cosine_sim(leona_alistar_comp, leona_nautilus_comp) > \
       cosine_sim(leona_alistar_comp, poke_comp)

# 검증 2: Pre-game 승률 예측 정확도 (Gate B)
# K-NN with comp_vector → AUC > 0.55 달성 여부
# 현재: composition_embeddings 1,572 nodes 적재 완료 / diskann 인덱스 미빌드
```

---

## 패치 대응 전략

패치마다 챔피언 수치가 변경됨 → 임베딩 자동 재계산 필요

```python
# 패치 감지 → 재임베딩 트리거
# Data Dragon 버전 변경 감지 → Lambda 실행
# → champion_feature_vector 재계산
# → composition_embeddings 테이블 재생성 (match_id 기준)
# → diskann 인덱스 재빌드

# 소요 시간 추정:
# - 챔피언 피처 재계산: < 1초
# - 100만 경기 재임베딩: ~30분 (배치)
# - diskann 인덱스 재빌드: ~10분
```

---

## 관련 파일

- [[_index]] — 프로젝트 전체 개요 (Neo4j 정적 온톨로지 · composition_embeddings DDL 포함)
- [[architecture-mermaid]] — 현재 구현 아키텍처 (§1 Gate A)

> **참고**: 목표 아키텍처(AWS)에서는 Amazon Neptune + pgvectorscale 사용 예정.
> Gate A 로컬 프로토타입은 **Neo4j (Docker)** + **TimescaleDB composition_embeddings 테이블** 사용.
