---
type: note
date: 2026-04-30
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [architecture, mermaid, diagram]
---

# 아키텍처 다이어그램 (Mermaid)

> 인터랙티브 HTML 버전: [[architecture-diagram]]
> 기술 상세: [[_index]]
>
> **문서 구성**: §1 현재 구현 (로컬 프로토타입, Gate A 완료) → §2 목표 아키텍처 (AWS 프로덕션) → §3-5 세부 흐름

---

## 1. 현재 구현 아키텍처 (로컬 프로토타입 · 2026-04-30)

> Gate A 완료 기준. AWS 인프라 없이 로컬에서 전체 파이프라인 검증 완료.

```mermaid
flowchart TD
    subgraph SRC["📡 데이터 소스"]
        A1["Riot Esports\nData API\n(경기 목록·결과)"]
        A2["spectator-v5\nAPI (메타)"]
        A4["Data Dragon\n챔피언·아이템 메타"]
    end

    subgraph COL["⚡ 수집 레이어 (로컬)"]
        B1["LolesportsLivePoller\nwindow + details\n병렬 폴링"]
        B2["Match-v5 Timeline\n과거 경기 캐시\n2,000경기"]
    end

    subgraph STORE["💾 저장 레이어 (로컬 Docker)"]
        subgraph PG["TimescaleDB (PostgreSQL 16)"]
            C1["team_state\n2,000경기 / 106,566행\nwin_prob·model_signal"]
            C2["teamfight_events\n8,711건"]
            C3["canonical_games\n541경기"]
            C4["composition_embeddings\n1,572 nodes (PCA 64dim)"]
        end
        C5["Neo4j\n정적 온톨로지\n챔피언·선수·팀"]
    end

    subgraph ML["🧠 모델 레이어 (로컬)"]
        D1["WinProbPredictor\nEXP004 XGBoost (.ubj)\n폴백: EXP003 LightGBM\nAUC 0.8353"]
        D2["TeamfightDetector\nRule-based\n(LSTM 미구현)\nAcc 83.4%"]
        D3["CompositionEmbedding\nPCA 64dim\n(K-NN 미구현)"]
    end

    subgraph RT["🔄 실시간 처리"]
        E1["LiveStatsReceiver\non_frame() → WinProb\nFightWindow 관리"]
        E2["Bedrock Commentary\nmock 모드\n(실 크레덴셜 미연결)"]
    end

    subgraph OUT["📊 출력"]
        F1["win_prob_swings.csv\nTop 50 스윙 경기"]
        F2["콘솔 해설 출력\nLevel 1~4 (mock)"]
    end

    A1 & A2 --> B1
    A1 --> B2
    B1 --> E1
    B2 --> C1 & C2
    A4 --> C5

    C5 --> D3
    D3 --> C4

    E1 --> D1 & D2
    D1 --> C1
    D2 --> C2

    E1 --> E2
    E2 --> F2
    C1 --> F1

    classDef done fill:#0f2318,stroke:#10b981,stroke-width:2px,color:#6ee7b7
    classDef mock fill:#1f1a0a,stroke:#f59e0b,stroke-width:1.5px,color:#fcd34d
    classDef pending fill:#1a1a1a,stroke:#6b7280,stroke-width:1px,color:#9ca3af

    class C1,C2,C3,C4,C5,D1,D2,D3,E1,F1 done
    class E2,F2 mock
```

**범례**: 🟢 구현 완료 · 🟡 mock 모드 · ⚫ 미구현

---

## 2. 목표 아키텍처 (클라우드 프로덕션 · 미래 상태)

> **주의**: 이 다이어그램은 Gate D 이후 클라우드 프로덕션 목표 상태. 현재 로컬 프로토타입 현황은 §1 참조.
> LSTM·K-NN·DynamoDB·Kinesis·SageMaker는 Gate B/C 이후 구현 예정.
> **그래프 DB**: Amazon Neptune 아님 → **Neo4j AuraDB** (Bolt 호환, 코드 변경 없이 이전 가능, GDS 알고리즘 지원)

```mermaid
flowchart TD
    subgraph SRC["📡 데이터 소스"]
        A1["Riot Esports\nData API"]
        A2["spectator-v5\nAPI"]
        A4["Data Dragon\n챔피언·아이템 메타"]
    end

    subgraph ING["⚡ 수집 레이어 (AWS)"]
        B1["API Gateway\n+ Lambda Ingestion"]
        B2["Kinesis\nData Streams"]
        B3["Lambda\nParser"]
        B4["Lambda\nEvent Detector"]
    end

    subgraph STORE["💾 저장 레이어"]
        subgraph PG["TimescaleDB + pgvectorscale (동일 PostgreSQL 인스턴스)"]
            direction LR
            C1["📊 시계열 테이블\nplayer_state · game_events\nteam_state · fight_events"]
            C2["🔷 벡터 테이블\nstate_embedding 128dim\ncomp_embedding 64dim\nfight_embedding 128dim"]
        end
        C3["DynamoDB\n현재 스냅샷"]
        C4["Neo4j AuraDB\n정적 온톨로지\n챔피언-아이템-팀-선수\n+ 선수 에이전트 그래프"]
        C5["S3\n정적 메타 버저닝"]
    end

    subgraph ML["🧠 추론 레이어 (SageMaker)"]
        D1["LSTM\n한타 예측\n30초 슬라이딩 윈도우\nP(fight) 출력"]
        D2["LightGBM\nWin Probability\n10초 인터벌"]
        D3["XGBoost\nTeamfight Outcome\n한타 직전 이진분류"]
        D4["pgvectorscale\nK-NN 검색\n유사 상황 empirical prob"]
    end

    subgraph ENS["⚖️ 앙상블 · 트리거"]
        E0["앙상블\n0.5×LSTM + 0.3×K-NN\n+ 0.2×Rule"]
        E1["EventBridge\nP > 0.75 트리거\nWin Prob 급변 감지"]
    end

    subgraph GEN["✨ 생성 레이어"]
        F1["Lambda\n관전 포인트 계산\nInformation Density\nRAG 컨텍스트 구성"]
        F2["Amazon Bedrock\n유저 수준별 해설\nLevel 1~4"]
        F3["WebSocket\nAPI Gateway"]
    end

    subgraph CLIENT["📱 클라이언트"]
        G1["오버레이 / 앱\n• 관전 포인트 알림\n• Win Probability 그래프\n• 한타 딥다이브 해설\n• 경기 결정 한타 TOP 3"]
    end

    subgraph STATIC["🗂️ 정적 파이프라인"]
        S1["S3 패치 데이터"]
        S2["Lambda ETL"]
        S3["Neo4j AuraDB 갱신"]
        S4["SageMaker\nFeature Store"]
    end

    %% 수집 흐름
    A1 & A2 --> B1
    B1 --> B2
    B2 --> B3 & B4
    B3 --> C1 & C2
    B4 --> C1 & C3

    %% 정적 파이프라인
    A4 --> S1 --> S2 --> S3 & S4
    S3 --> C4

    %% 추론 흐름
    C1 & C4 --> D1 & D2 & D3
    C2 --> D4
    C1 & C2 --> D2

    %% 앙상블
    D1 & D4 --> E0
    E0 & D2 & D3 --> E1

    %% 생성 흐름
    E1 --> F1
    C2 --> F1
    F1 --> F2
    F2 --> F3
    F3 --> G1

    %% 스타일
    classDef srcStyle fill:#1a1f2e,stroke:#3b82f6,stroke-width:1.5px,color:#93c5fd
    classDef ingStyle fill:#1a1f2e,stroke:#6366f1,stroke-width:1.5px,color:#a5b4fc
    classDef tsStyle fill:#0f2318,stroke:#10b981,stroke-width:2px,color:#6ee7b7
    classDef vecStyle fill:#1a0f2e,stroke:#8b5cf6,stroke-width:2px,color:#c4b5fd
    classDef nepStyle fill:#1a1a0f,stroke:#f59e0b,stroke-width:1.5px,color:#fcd34d
    classDef mlStyle fill:#1f1a0a,stroke:#f59e0b,stroke-width:1.5px,color:#fcd34d
    classDef genStyle fill:#1a0f18,stroke:#ec4899,stroke-width:1.5px,color:#f9a8d4
    classDef clientStyle fill:#0a1a1a,stroke:#06b6d4,stroke-width:2px,color:#67e8f9

    class A1,A2,A3,A4 srcStyle
    class B1,B2,B3,B4 ingStyle
    class C1 tsStyle
    class C2,D4 vecStyle
    class C3,C4,C5,S1,S2,S3,S4 nepStyle
    class D1,D2,D3,E0,E1 mlStyle
    class F1,F2,F3 genStyle
    class G1 clientStyle
```

---

## 3. 온톨로지 레이어 구조

```mermaid
block-beta
  columns 1

  block:L3["Layer 3 : 추론·생성 레이어"]:1
    columns 4
    lstm["LSTM\n한타 예측"]
    lgbm["LightGBM\nWin Probability"]
    xgb["XGBoost\nTeamfight Outcome"]
    bedrock["Bedrock\n해설 생성"]
  end

  space

  block:L2B["Layer 2B : 벡터 레이어 (pgvectorscale)"]:1
    columns 3
    sv["게임 상태\n벡터 128dim"]
    cv["챔피언 구성\n벡터 64dim"]
    fv["한타 컨텍스트\n벡터 128dim"]
  end

  block:L2A["Layer 2A : 동적 온톨로지 (TimescaleDB)"]:1
    columns 4
    ps["player_state\n1초 폴링"]
    ge["game_events\n이벤트 기반"]
    ts["team_state\n5초 폴링"]
    fe["fight_events\n한타 감지 시"]
  end

  space

  block:L1["Layer 1 : 정적 온톨로지 (Neo4j AuraDB)"]:1
    columns 4
    ch["Champion\n챔피언·스킬"]
    it["Item\n아이템 관계"]
    pl["Player\n선수 프로파일\n+ 행동 에이전트(Gate C)"]
    tm["Team\n팀·로스터"]
  end
```

---

## 4. 한타 예측 앙상블 흐름

```mermaid
flowchart LR
    GS["게임 상태\n(1초 단위)"]

    GS --> R1
    GS --> R2
    GS --> R3
    GS --> R4

    R1["Rule Engine\n공간 클러스터링\n오브젝트 근접\n시야 파괴"]
    R2["LSTM\n30초 윈도우\n시계열 분류"]
    R3["pgvectorscale\nTOP-20 유사 상황\n실제 한타 발생률"]
    R4["Neo4j AuraDB 피처\nengage_potential\nCC체인 가능 여부"]

    R1 -->|rule_score| ENS
    R2 -->|P_lstm| ENS
    R3 -->|P_empirical| ENS
    R4 -->|onto_score| ENS

    ENS["앙상블\n0.5×P_lstm\n+ 0.3×P_empirical\n+ 0.2×rule_score"]

    ENS -->|P < 0.75| WAIT["대기\n계속 모니터링"]
    ENS -->|P ≥ 0.75| ALERT["🔔 한타 예측 발동\n관전 포인트 계산\nRAG 해설 준비"]
```

---

## 5. 서비스 흐름 분리 — Flow A vs Flow B

> Gate C부터 두 흐름이 병렬로 운영됩니다.

```mermaid
flowchart TD
    subgraph FA["Flow A — 실시간 해설 (이벤트 기반)"]
        direction LR
        EV["한타 이벤트\n발생"] --> RC["RAG 컨텍스트\n수집\n(Neo4j 정적 쿼리)"]
        RC --> BD_A["Bedrock Claude\nHaiku: L1/L2\nSonnet: L3/L4"]
        BD_A --> WS["WebSocket\n→ 클라이언트"]
    end

    subgraph FB["Flow B — 쿼리 인터페이스 (사용자 질의)"]
        direction LR
        UQ["사용자\n자연어 질문"] --> GR["neo4j-graphrag-python\nText2CypherRetriever\n(스키마 + 예시 10개 캐싱)"]
        GR --> BD_B["Bedrock Claude Haiku\nCypher 생성"]
        BD_B --> NEO["Neo4j 실행"]
        NEO --> BD_C["Bedrock Claude Sonnet\n자연어 답변"]
        BD_C --> ANS["답변 반환"]
    end

    subgraph NC["NeoConverse (내부 탐색 전용)"]
        direction LR
        EXP["Gate C 내부\n질의 패턴 파악"] --> EXTRACT["자주 나오는 패턴\n→ EXAMPLES 추가"]
        EXTRACT --> FB
    end
```

**핵심 결정:**
- NeoConverse는 프로덕션 서비스에 사용하지 않음 (Labs 실험 프로젝트)
- Flow B 프로덕션 구현: `neo4j-graphrag-python` + Bedrock 커스텀 래퍼
- 팀 구현 부담: 도메인 예시 10개 + 래퍼 ~60줄

---

## 7. 승부 예측 3단계 흐름

```mermaid
flowchart TD
    PG["Pre-game\n드래프트 완료"]
    IG["In-game\n10초 간격"]
    TF["Teamfight\n직전 상태"]

    PG -->|"챔피언 구성 벡터\n선수 폼 지표\nH2H 기록"| M1
    IG -->|"골드차·킬차\n타워·오브젝트\n시간"| M2
    TF -->|"HP%·스킬준비율\nengage_potential\n포지션"| M3

    M1["K-NN + GBM\n구성 유사도"]
    M2["LightGBM\nWin Probability"]
    M3["XGBoost\n이진분류"]

    M1 -->|"팀A 58% 예상"| OUT1["경기 시작 전\n예상 승률 표시"]
    M2 -->|"10초마다 갱신"| OUT2["Win Prob 그래프\n실시간 업데이트"]
    M3 -->|"팀A 62% 우세"| OUT3["한타 오버레이\n실시간 확률 표시"]

    OUT2 --> SWING["teamfight_value\n= Prob_after - Prob_before\n→ 경기 결정 한타 TOP 3 산출"]
```

---

## 연결 문서

- [[_index]] — 기술 아키텍처 전체 (온톨로지 레이어 · DB 설계 · SQL)
- [[service-scenario]] — 서비스 구조 · 유저 여정 · 기능 매핑
- [[architecture-diagram]] — 인터랙티브 HTML 다이어그램
