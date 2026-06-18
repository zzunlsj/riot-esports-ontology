---
type: research
date: 2026-04-30
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [Neptune, openCypher, graph, schema, ontology]
---

# Neptune openCypher 스키마 초안

> Layer 1 온톨로지의 Amazon Neptune 구현 스키마.
> openCypher 기반 Property Graph 모델.
>
---

## §0. 현재 구현 상태 (2026-04-30 기준)

| 항목 | 상태 | 비고 |
|------|------|------|
| DB 엔진 | **Neo4j Desktop** (bolt://localhost:7687) | Neptune 이전은 Gate B 이후 |
| Champion 노드 | ✅ 172개 적재 | DDragon 기반 |
| ChampionPatchChange | ✅ 280건 | 패치 diff 적재 |
| Player / Team / Tournament / Series / Game | ✅ 적재 완료 | LCK 2025 시즌 기준 |
| PlayerGameStats | ✅ 5,410개 | 685경기 |
| Composition Embeddings | ✅ 1,572 nodes / PCA 64dim | pgvectorscale Layer 2B 적재 완료, diskann 인덱스 Gate B |
| COUNTERS / SYNERGIZES_WITH | ⚫ Phase 2 미구현 | lolalytics 크롤 선행 필요 |
| Ability / Item / MapZone / Objective | ⚫ Phase 2 미구현 | DDragon 파싱 선행 필요 |
| CHAINS_WITH / HAS_ABILITY | ⚫ Phase 2 미구현 | Ability 노드 선행 필요 |
| TeamfightRecord 노드 + 관계 | ⚫ Gate B | Live Stats 실 연결 이후 |
| Neptune 이전 | ⚫ Gate B 이후 | 로컬 프로토타입 검증 완료 후 이전 |

---

> **"정적"의 의미**: 경기 중에는 불변, 경기 사이에 업데이트됨.
> 선수 이적·챔피언 패치처럼 주기적·이산적으로 바뀌는 데이터는 Neptune에서 관리.
> 초 단위로 변하는 HP·쿨타임·위치는 TimescaleDB(Layer 2A) 담당.
> → 동적 변화 관리 패턴: [섹션 6 참조](#6-동적-온톨로지-변화-관리)
>
> 데이터 품질 운영 원칙:
> Neptune canonical 속성은 Riot API, lolesports, Data Dragon 같은 공식 소스 기반 데이터만 사용한다.
> 비공식 보조 소스는 현재 운영 경로에서 사용하지 않는다.

---

## 노드(Node) 스키마

### Champion

```cypher
CREATE (:Champion {
  id:               'Jinx',           // string  — Data Dragon key
  championId:       222,              // int     — Riot numeric ID
  name:             'Jinx',
  title:            'the Loose Cannon',
  role:             ['bot'],          // list<string>
  class:            'marksman',       // assassin|fighter|mage|marksman|support|tank
  attackRange:      525,              // int  — <300 melee, >=300 ranged
  scalingType:      'AD',             // AD|AP|hybrid|tank
  engagePotential:  'low',            // low|medium|high
  ccTypes:          ['slow'],         // list<string>
  baseHp:           610,
  baseArmor:        21,
  baseMagicResist:  30,
  baseAttackDamage: 57,
  baseMovementSpeed: 325,
  patchVersion:     '15.7'
})
```

### Ability

```cypher
CREATE (:Ability {
  id:             'Jinx_Q',
  championId:     'Jinx',
  slot:           'Q',               // Q|W|E|R|Passive
  name:           'Switcheroo!',
  cooldownBase:   [10, 10, 10, 10, 10],  // list<int> 레벨별
  cooldownWithMaxCdr: [5, 5, 5, 5, 5],
  castRange:      0,                 // 0 = 자기자신
  targetType:     'self',            // point|direction|unit|aoe|global|self
  damageType:     'physical',        // physical|magic|true|none
  ccType:         null,
  isSkillshot:    false,
  isChanneled:    false,
  interruptible:  false,
  comboRole:      'execute'          // initiator|follow-up|peel|execute|sustain
})
```

### Item

```cypher
CREATE (:Item {
  itemId:         3031,
  name:           'Infinity Edge',
  cost:           3400,
  tags:           ['Damage', 'CriticalStrike'],
  stat_ad:        70,
  stat_critChance: 20,
  buildPath:      [1038, 1037, 1042],  // list<int>
  isFinished:     true,
  patchVersion:   '15.7'
})
```

### Player

```cypher
CREATE (:Player {
  puuid:          'xxxxxxxxxxx...78chars',
  riotId:         'Faker#KR1',
  gameName:       'Faker',
  tagLine:        'KR1',
  tier:           'CHALLENGER',
  lp:             1200,
  mainRoles:      ['mid'],
  playstyle:      'aggressive',       // aggressive|defensive|farm|roam
  region:         'KR',
  updatedAt:      '2026-03-23'
})
```

### Team

```cypher
CREATE (:Team {
  id:             'T1',
  name:           'T1',
  region:         'LCK',
  earlyGameFocus:       0.8,
  objectivePriority:    0.9,
  teamfightComposition: 0.85,
  splitPushTendency:    0.2,
  updatedAt:      '2026-03-23'
})
```

### MapZone

```cypher
CREATE (:MapZone {
  id:              'baron-pit',
  name:            'Baron Nashor Pit',
  xMin:            3800, xMax: 5000,
  yMin:            9800, yMax: 11000,
  strategicWeight: 0.95              // 20분 이후 최고조
})
```

### Objective

```cypher
CREATE (:Objective {
  id:              'baron',
  type:            'baron',          // dragon|baron|herald|tower|inhibitor|nexus
  spawnTime:       1200,             // 초 (20분)
  respawnInterval: 360,              // 6분
  buffEffect:      'Baron Empowerment: minion buff + AD/AP +40',
  strategicValue:  0.95
})
```

---

## 관계(Relationship) 스키마

### COUNTERS — 챔피언 카운터 관계

```cypher
(:Champion)-[:COUNTERS {
  strength:     0.8,              // 0.0~1.0  카운터 강도
  context:      'laning',         // laning|teamfight|scaling
  patchVersion: '15.7',
  source:       'lolalytics',     // 데이터 출처
  sampleSize:   18420,            // 표본 수
  winRateDelta: 0.041,            // 상대 대비 승률 차이
  goldDiffAt15: 312,              // 라인전 우위 지표
  roleContext:  'support-vs-support',
  sourceType:   'derived'         // curated|derived
}]->(:Champion)

-- 예시
MATCH (a:Champion {id:'Leona'}), (b:Champion {id:'Soraka'})
CREATE (a)-[:COUNTERS {strength:0.85, context:'laning'}]->(b)
```

### SYNERGIZES_WITH — 챔피언 시너지

```cypher
(:Champion)-[:SYNERGIZES_WITH {
  strength:   0.9,
  context:    'teamfight',
  combo:      'Leona E → Orianna R',
  patchVersion: '15.7',
  sampleSize:  9250,
  winRateDelta: 0.036,
  sourceType:  'derived'
}]->(:Champion)
```

### 계산된 관계 생성 원칙

`COUNTERS`, `SYNERGIZES_WITH`, `MAINS` 같은 관계는 Neptune/Neo4j가 직접 계산하는 것이 아니라,
외부 통계 또는 과거 경기 데이터에서 집계한 결과를 **관계 지식으로 승격**해 저장한다.

정리:

- `TimescaleDB` / pandas / SQL
  - raw timeline, event, frame 집계
  - matchup win rate, gold diff@15, teamfight success rate 계산
- `Neo4j`
  - 계산된 관계를 patch/version/context 단위로 저장
  - 다중 홉 탐색과 설명 가능한 질의에 사용

권장 저장 규칙:

- 원천 시계열 데이터는 Neo4j에 넣지 않는다
- 관계 강도는 `strength`로 정규화한다
- 통계 신뢰도는 `sampleSize`와 `winRateDelta`로 남긴다
- 패치에 따라 값이 바뀌는 관계는 반드시 `patchVersion`을 속성으로 둔다
- 계산 결과임을 표시하기 위해 `sourceType='derived'`를 둔다

예시 파이프라인:

1. TimescaleDB에서 patch별/role별 matchup 집계
2. Python에서 strength 정규화
3. Neptune openCypher로 `COUNTERS` / `SYNERGIZES_WITH` 업서트

```cypher
MATCH (a:Champion {id: $src}), (b:Champion {id: $dst})
MERGE (a)-[r:COUNTERS {patchVersion: $patch, context: $context, roleContext: $roleContext}]->(b)
SET r.strength     = $strength,
    r.sampleSize   = $sampleSize,
    r.winRateDelta = $winRateDelta,
    r.goldDiffAt15 = $goldDiffAt15,
    r.source       = $source,
    r.sourceType   = 'derived',
    r.updatedAt    = $updatedAt
```

### HAS_ABILITY — 챔피언-스킬 보유

```cypher
(:Champion)-[:HAS_ABILITY {slot: 'Q'}]->(:Ability)
```

### CHAINS_WITH — CC 콤보 체인

```cypher
(:Ability)-[:CHAINS_WITH {
  chainType:    'hard_cc_follow',   // hard_cc_follow|interrupt|interrupt_channel
  windowSec:    2.0,                // 연계 가능 시간 윈도우
  reliability:  'high'
}]->(:Ability)

-- 예시: Leona E(knockup) → Orianna R(AOE)
MATCH (a:Ability {id:'Leona_E'}), (b:Ability {id:'Orianna_R'})
CREATE (a)-[:CHAINS_WITH {chainType:'hard_cc_follow', windowSec:1.5}]->(b)
```

### BUILDS_INTO — 아이템 빌드 트리

```cypher
(:Item)-[:BUILDS_INTO {goldEfficiency: 0.92}]->(:Item)
```

### CORE_ITEM_FOR — 챔피언 핵심 아이템

```cypher
(:Item)-[:CORE_ITEM_FOR {
  priority:     1,                  // 1=first item, 2=second, ...
  winRateDelta: +0.03,
  pickRate:     0.72
}]->(:Champion)
```

### MAINS — 선수 주력 챔피언

```cypher
(:Player)-[:MAINS {
  gamesPlayed:  87,
  winRate:      0.61,
  kda:          4.2,
  lastPlayed:   '2026-03-20'
}]->(:Champion)
```

### MEMBER_OF — 팀 소속 (이적 이력 추적)

> 선수 이적 시 기존 관계를 삭제하지 않고 `endDate`를 기록.
> 현재 소속은 `endDate IS NULL`로 식별.

```cypher
(:Player)-[:MEMBER_OF {
  role:       'mid',
  jerseyNum:  null,
  startDate:  '2022-01-01',   // 입단일
  endDate:    null,            // null = 현재 소속, 이적 시 날짜 기록
  isStarter:  true,
  contractEnd: '2026-12-31'
}]->(:Team)
```

#### 이적 처리 쿼리 (이적 발생 시 Lambda 실행)

```cypher
-- ① 기존 소속 종료
MATCH (p:Player {puuid: $puuid})-[r:MEMBER_OF]->(t:Team)
WHERE r.endDate IS NULL
SET r.endDate = $transferDate, r.isStarter = false

-- ② 새 팀 소속 생성
MATCH (p:Player {puuid: $puuid}), (newTeam:Team {id: $newTeamId})
CREATE (p)-[:MEMBER_OF {
  role:      $role,
  startDate: $transferDate,
  endDate:   null,
  isStarter: true
}]->(newTeam)
```

#### 날짜 기준 소속 조회

```cypher
-- "2026-02-01 기준 T1 로스터는?"
MATCH (p:Player)-[r:MEMBER_OF]->(t:Team {id: 'T1'})
WHERE r.startDate <= '2026-02-01'
  AND (r.endDate IS NULL OR r.endDate >= '2026-02-01')
RETURN p.riotId, r.role, r.startDate
ORDER BY r.role
```

### LOCATED_IN — 오브젝트 위치

```cypher
(:Objective)-[:LOCATED_IN]->(:MapZone)
```

### ADJACENT_TO — 구역 인접성

```cypher
(:MapZone)-[:ADJACENT_TO {
  transitionTime: 8    // 도달 평균 이동 시간 (초)
}]->(:MapZone)
```

---

## 핵심 openCypher 쿼리 예시

### Q1. 현재 조합에서 적 조합을 가장 잘 카운터하는 챔피언은?

```cypher
MATCH (myChamp:Champion)<-[:COUNTERS]-(counter:Champion)-[:COUNTERS]->(enemyChamp:Champion)
WHERE myChamp.id IN $myTeamChampions
  AND enemyChamp.id IN $enemyTeamChampions
RETURN counter.id,
       count(*) AS counterCount,
       avg(r.strength) AS avgStrength
ORDER BY counterCount DESC, avgStrength DESC
LIMIT 3
```

### Q2. 현재 조합의 CC 체인 가능 콤보 탐색

```cypher
MATCH (initiator:Champion)-[:HAS_ABILITY]->(initAbility:Ability)-[:CHAINS_WITH]->(followAbility:Ability)<-[:HAS_ABILITY]-(follower:Champion)
WHERE initiator.id IN $myTeam
  AND follower.id IN $myTeam
  AND initiator.id <> follower.id
  AND initAbility.comboRole = 'initiator'
RETURN initiator.id, initAbility.name,
       follower.id, followAbility.name,
       r.windowSec
ORDER BY r.reliability DESC
```

### Q3. 특정 선수의 현재 챔피언풀에서 상대 조합을 카운터하는 챔피언

```cypher
MATCH (player:Player {riotId: $playerId})-[m:MAINS]->(champ:Champion)-[c:COUNTERS]->(enemyChamp:Champion)
WHERE enemyChamp.id IN $enemyTeam
  AND m.gamesPlayed > 20
RETURN champ.id, m.winRate, c.strength
ORDER BY m.winRate * c.strength DESC
LIMIT 5
```

### Q4. 바론 근처 구역 인접 맵존 탐색 (이동 경로 분석)

```cypher
MATCH path = (start:MapZone {id:'baron-pit'})-[:ADJACENT_TO*1..2]->(zone:MapZone)
RETURN zone.id, length(path) AS hops,
       reduce(t = 0, r IN relationships(path) | t + r.transitionTime) AS estimatedTravelSec
ORDER BY hops, estimatedTravelSec
```

---

## 인덱스 설계

```cypher
-- 챔피언 ID 인덱스 (가장 빈번한 조회)
CREATE INDEX champion_id FOR (c:Champion) ON (c.id)
CREATE INDEX champion_numeric_id FOR (c:Champion) ON (c.championId)

-- 선수 puuid 인덱스 (API 매핑용)
CREATE INDEX player_puuid FOR (p:Player) ON (p.puuid)
CREATE INDEX player_riotid FOR (p:Player) ON (p.riotId)

-- 아이템 ID 인덱스
CREATE INDEX item_id FOR (i:Item) ON (i.itemId)
```

---

## 데이터 적재 전략

| 소스 | 대상 노드/관계 | 갱신 주기 | 변화 유형 |
|------|-------------|---------|---------|
| Data Dragon JSON | Champion, Ability, Item | 패치마다 (2주) | 속성 덮어쓰기 + S3 아카이빙 |
| lolalytics.com 크롤 | COUNTERS, SYNERGIZES_WITH | 주 1회 | 엣지 가중치 갱신 |
| core_item 통계 | CORE_ITEM_FOR | 주 1회 | 엣지 속성 갱신 |
| Riot API (match-v5) | Player, MAINS | 경기 종료 후 | 집계 통계 누적 |
| riotesportsdata.com | Team, MEMBER_OF | 로스터 변경 즉시 | **이적 처리 (startDate/endDate)** |

### Lambda ETL 파이프라인

```
[EventBridge 규칙]
  ├── 패치 감지 (Data Dragon 버전 변경) ─────────→ patch_updater Lambda
  ├── 주 1회 (일요일 03:00 KST) ────────────────→ stats_updater Lambda
  └── 로스터 변경 감지 ────────────────────────→ roster_updater Lambda

patch_updater Lambda:
  S3 (Data Dragon ZIP)
      ↓
  Champion/Ability/Item 속성 SET (patchVersion 갱신)
      ↓
  COUNTERS 관계 invalidate → lolalytics 재크롤 트리거
      ↓
  composition_embeddings 재생성 트리거 (~30분 소요)
      ↓
  S3: 이전 패치 JSON 아카이빙 (s3://bucket/patches/{version}.json)

roster_updater Lambda:
  riotesportsdata.com 로스터 페이지 감지
      ↓
  MEMBER_OF endDate 기록 + 새 관계 생성
      ↓
  Neptune Streams → DynamoDB 현재 스냅샷 갱신

stats_updater Lambda:
  lolalytics.com 크롤 (COUNTERS 승률 통계)
      ↓
  Neptune COUNTERS/SYNERGIZES_WITH 엣지 strength 갱신
      ↓
  CORE_ITEM_FOR winRateDelta, pickRate 갱신
```

---

## 6. 동적 온톨로지 변화 관리

> Neptune은 "영구 불변 그래프"가 아니라 **"현재 시점의 정규화된 지식 그래프"**로 운용.
> 변화의 성격에 따라 두 가지 패턴을 적용.

### 변화 유형별 처리 전략

| 변화 유형 | 예시 | 전략 | 이력 보존 |
|---------|------|------|---------|
| 선수 이적 | Faker가 T1 → GenG | 관계 타임스탬핑 | Neptune 내 (`startDate/endDate`) |
| 챔피언 패치 | Jinx Q 쿨타임 10→8초 | 속성 덮어쓰기 | S3 JSON 아카이빙 |
| 아이템 삭제 | 삭제된 아이템 | 노드에 `deprecated: true` 플래그 | Neptune 보존 |
| 카운터 통계 변동 | 패치 후 승률 변화 | 엣지 strength 값 갱신 | 갱신만, 이전값 미보존 |
| 팀 해체/창단 | 신규 팀 LCK 진입 | 신규 Team 노드 생성 | 영구 보존 |

---

### 패턴 A: 관계 타임스탬핑 (선수 이적)

가장 중요한 동적 관계. `MEMBER_OF` 엣지에 `startDate`/`endDate`를 두고 현재 소속은
`endDate IS NULL`로 식별. 이적 이력이 Neptune 내에 완전히 보존됨.

```cypher
-- 현재 활성 선수 목록 (이적 이력 제외)
MATCH (p:Player)-[r:MEMBER_OF]->(t:Team)
WHERE r.endDate IS NULL
RETURN t.id AS team, collect(p.riotId) AS currentRoster

-- Faker의 전체 이적 이력
MATCH (p:Player {riotId: 'Faker#KR1'})-[r:MEMBER_OF]->(t:Team)
RETURN t.id, r.startDate, r.endDate
ORDER BY r.startDate
```

---

### 패턴 B: 속성 덮어쓰기 + S3 아카이빙 (챔피언 패치)

챔피언·스킬 노드는 항상 최신 패치 기준 값을 유지.
이전 패치 값이 필요한 경우(과거 경기 해설 등) S3에서 로드.

```cypher
-- 패치 14.9 적용: Jinx Q 쿨타임 수정
MATCH (a:Ability {id: 'Jinx_Q'})
SET a.cooldownBase = [8, 8, 8, 8, 8],   -- 10 → 8초로 버프
    a.patchVersion = '14.9',
    a.updatedAt    = '2026-03-20'

-- 패치 14.9 적용: Jinx 기본 체력 수정
MATCH (c:Champion {id: 'Jinx'})
SET c.baseHp      = 640,               -- 610 → 640
    c.patchVersion = '14.9',
    c.updatedAt    = '2026-03-20'
```

S3 아카이빙 경로:
```
s3://{bucket}/patches/
├── 14.7/champions.json
├── 14.8/champions.json
└── 14.9/champions.json    ← 최신
```

---

### 패턴 C: deprecated 플래그 (아이템 삭제)

삭제된 아이템 노드는 제거하지 않고 플래그 처리.
과거 경기 해설에서 "당시 아이템" 참조 가능.

```cypher
-- Luden's Tempest이 다음 패치에서 삭제될 때
MATCH (i:Item {itemId: 3285})
SET i.deprecated    = true,
    i.deprecatedAt  = '2026-04-03',
    i.patchVersion  = '14.10'

-- 현재 구매 가능한 아이템만 조회 시 필터
MATCH (i:Item)-[:CORE_ITEM_FOR]->(c:Champion {id: 'Lux'})
WHERE i.deprecated IS NULL OR i.deprecated = false
RETURN i.name, i.cost ORDER BY i.cost
```

---

### Neptune vs. TimescaleDB 역할 경계

```
Neptune (Layer 1) — 주기적 업데이트
  ├── 챔피언 스탯·스킬 (2주 간격 패치)
  ├── 카운터·시너지 관계 (주 1회 통계 갱신)
  ├── 선수 소속 이력 (이적 즉시)
  └── 팀 스타일 지수 (월 1회 또는 시즌별)

TimescaleDB (Layer 2A) — 실시간 업데이트
  ├── 선수 현재 HP·마나·쿨타임 (1초 간격)
  ├── 경기 중 골드·레벨·위치 (1초 간격)
  └── 한타 이벤트 스트림 (즉시)

기준: "경기 중에 바뀌는가?" → YES → TimescaleDB / NO → Neptune
```

---

## 7. 실제 구현된 스키마 (현재 상태)

> **현재 DB**: Neo4j Desktop (로컬 bolt://localhost:7687) — Neptune 이전 전 로컬 프로토타입.
> 스키마 구조는 Neptune openCypher와 동일하게 유지. 이전 시 별도 마이그레이션 불필요.
>
> 위 섹션의 Ability/Item/MapZone/Objective 및 COUNTERS/SYNERGIZES_WITH 등은 Phase 2 예정.

### 실제 구현 노드

| 레이블 | 주요 속성 | 데이터 소스 |
|--------|---------|-----------|
| `Player` | `puuid`, `lolesportsId`, `riotId`, `gameName`, `tagLine`, `mainRoles`, `residency` | lolesports getTeams + Riot Account API |
| `Team` | `id`, `name`, `region`, `code` | lolesports getTeams |
| `Tournament` | `id`, `slug`, `startDate`, `endDate`, `league` | lolesports getTournamentsForLeague |
| `Series` | `id`, `date`, `block`, `format`, `formatCount`, `tournamentId` | lolesports getSchedule + getEventDetails |
| `Game` | `id`, `number`, `patch`, `date`, `winner`, `loser`, `winnerSide`, `durationSec`, `goldDiff`, `blueDragons`, `redDragons`, `blueBarons`, `redBarons`, `blueKills`, `redKills` | lolesports live-stats window + ending window |
| `Champion` | `id`, `name`, `title`, `tags`, `resource`, `moveSpeed`, `attackRange`, 기타 DDragon 스탯 | DDragon API |
| `ChampionPatchChange` | `id`, `championId`, `patch`, `changeType` (buff/nerf/adjust/new), `statChanges` (JSON) | DDragon 버전 간 diff |
| `PlayerGameStats` | `id`, `gameId`, `playerId`, `championId`, `role`, `side`, `kills`, `deaths`, `assists`, `cs`, `gold`, `killParticipation`, `championDamageShare`, `wardsPlaced`, `wardsDestroyed`, `items`, `keystoneId`, `primaryRuneStyle`, `secondaryRuneStyle`, `skillOrder`, `skillMaxOrder` | lolesports live-stats details API |

### 실제 구현 관계

| 관계 | 방향 | 주요 속성 |
|------|------|---------|
| `MEMBER_OF` | Player → Team | `startDate`, `role` |
| `COMPETED_IN` | Team → Series | `outcome`, `gameWins` |
| `IN_TOURNAMENT` | Series → Tournament | — |
| `PART_OF` | Game → Series | — |
| `PLAYED_AS` | Player → Champion | `gameId`, `role`, `side` |
| `FEATURES` | Game → Champion | — |
| `HAD_STATS` | Player → PlayerGameStats | — |
| `IN_GAME` | PlayerGameStats → Game | — |
| `CHANGED_IN` | Champion → ChampionPatchChange | — |

### 주요 노드 속성 예시

```cypher
// Player 노드
(:Player {
  puuid:        'xxxxxxxxxxx...78chars',
  lolesportsId: '98767975951139628',   // gameMetadata.esportsPlayerId와 매핑
  riotId:       'T1 Faker',
  gameName:     'Faker',
  tagLine:      'KR1',
  mainRoles:    ['mid'],
  residency:    'KR',
  updatedAt:    '2026-03-27'
})

// Game 노드 (전체 속성)
(:Game {
  id:           '113503303285384356',
  number:       1,                     // 시리즈 내 게임 번호
  patch:        '15.7',
  date:         '2025-06-07T06:00:00Z',
  winner:       'T1',
  loser:        'Gen.G',
  winnerSide:   'blue',
  durationSec:  2805,                  // ~47분
  goldDiff:     4823,                  // blue - red (양수 = blue 우세)
  blueDragons:  ['infernal', 'hextech'],
  redDragons:   ['chemtech'],
  blueBarons:   1,
  redBarons:    0,
  blueKills:    15,
  redKills:     8
})

// PlayerGameStats 노드
(:PlayerGameStats {
  id:                   '113503303285384356_1',
  gameId:               '113503303285384356',
  playerId:             '98767975951139628',  // lolesportsId
  championId:           'Azir',
  role:                 'mid',
  side:                 'blue',
  level:                18,
  kills:                5,  deaths: 1,  assists: 9,
  cs:                   312,
  gold:                 18450,
  killParticipation:    0.7333,
  championDamageShare:  0.3012,
  wardsPlaced:          18,
  wardsDestroyed:       6,
  items:                '[3006, 3003, 3165, 3089, 3135, 3157]',
  keystoneId:           8230,          // Phase Rush
  primaryRuneStyle:     8200,          // Sorcery
  secondaryRuneStyle:   8300,          // Inspiration
  skillOrder:           'EQWQQRQEQEEREWWWRW',
  skillMaxOrder:        'QEW'          // Q→E→W 순으로 맥스
})
```

### 미구현 (Phase 2 예정)

| 항목 | 상태 | 비고 |
|------|------|------|
| `Ability` 노드 | Phase 2 | DDragon abilities 파싱 필요 |
| `Item` 노드 | Phase 2 | DDragon items.json |
| `MapZone` / `Objective` 노드 | Phase 2 | 정적 정의 |
| `COUNTERS` / `SYNERGIZES_WITH` | Phase 2 | lolalytics 크롤 |
| `HAS_ABILITY` / `CHAINS_WITH` | Phase 2 | Ability 노드 선행 |
| `BUILDS_INTO` / `CORE_ITEM_FOR` | Phase 2 | Item 노드 선행 |
| `MAINS` (Player → Champion) | Phase 2 | match-v5 집계 |
| `TeamfightRecord` 노드 + 관계 | **Gate B** | TimescaleDB `teamfight_events` 8,711건 확보 완료 — Neo4j 승격은 Live Stats 실 연결 후 |
| 밴 데이터 | Phase 2 | lolesports 공개 API 미제공, match-v5 필요 |
| tier / lp / playstyle (Player) | Phase 2 | Riot Ranked API |

---

## 8. 실시간 API 연결 시 온톨로지 확장 계획

> **⚠️ Gate B 이후 적용 예정**: 이 섹션의 내용은 Live Stats WebSocket 실 연결이 완료된 이후에 구현됨.
> 현재(Gate A 완료 기준) LiveStatsReceiver는 mock replay 모드로만 동작 중이며,
> TeamfightRecord 노드 생성 및 관련 관계 승격은 Gate B P0 과제임.
>
> 현재는 경기 종료 후 데이터만 수집. Riot Esports Live Stats API가 연결되면
> 경기 중 동적 상태가 실시간으로 들어오며 아래 확장이 가능해짐.

### 8-1. 추가될 노드/관계 (Neo4j Layer 1)

#### TeamfightRecord 노드

```cypher
(:TeamfightRecord {
  id:           'fight_{gameId}_{timestamp}',
  gameId:       '113503303285384356',
  gameTimeSec:  1823,           // 게임 내 발생 시점
  centroidX:    9200,           // 한타 중심 좌표
  centroidZ:    5400,
  durationSec:  12,             // 한타 지속 시간
  goldSwing:    1850,           // 한타로 인한 골드 변동 (blue - red)
  killsInFight: 3,
  objectiveConverted: 'dragon', // 한타 후 획득 오브젝트 (없으면 null)
  winnerSide:   'blue'
})
```

#### 추가 관계

```cypher
// 게임-한타 연결
(Game)-[:HAD_TEAMFIGHT]->(TeamfightRecord)

// 선수-한타 참전 이력
(Player)-[:PARTICIPATED_IN {
  outcome:     'win',      // win | loss
  kills:       2,
  deaths:      0,
  assists:     1,
  goldSwing:   450         // 해당 선수가 기여한 골드 변동
}]->(TeamfightRecord)
```

#### COUNTERS 관계 보정

기존 COUNTERS 관계는 lolalytics 통계(솔로랭크 기반)였지만, 실제 프로경기 한타 데이터로 `in_fight` context를 추가:

```cypher
(:Champion)-[:COUNTERS {
  strength:     0.78,
  context:      'in_fight',   // laning | teamfight | scaling → in_fight 추가
  patchVersion: '15.7',
  source:       'lck_2025'    // 실제 LCK 경기 데이터 기반
}]->(:Champion)
```

#### Player 노드 집계 속성 추가

```cypher
MATCH (p:Player)
SET p.avgFightKP       = 0.71,   // 한타 평균 킬 관여율
    p.avgFightGoldSwing = 320,    // 한타 기여 평균 골드 변동
    p.fightWinRate      = 0.63    // 한타 승률
```

---

### 8-2. TimescaleDB Retention 정책

> **"최신 데이터만 보유하라"는 권장사항은 `player_state` 테이블에만 적용.**
> `teamfight_events`와 `game_events`는 영구 보존 대상.

| 테이블 | Retention 권장 | 이유 |
|--------|--------------|------|
| `player_state` (1초 폴링) | **삭제 가능** — 경기 종료 후 요약본만 보존 | 초당 10행, 1경기 = 수만 행. 원본 불필요 |
| `game_events` (킬/오브젝트 이벤트) | **영구 보존** | 수량 적고 분석 가치 높음 |
| `teamfight_events` | **영구 보존** | 한타 비교 학습 데이터의 핵심 |
| `team_state` (5초 폴링) | **경기 종료 후 삭제 가능** | 한타 감지에 사용 후 `teamfight_events`로 요약 |

**결론**: 과거 한타 비교는 `teamfight_events` 테이블 + pgvectorscale 임베딩으로 유지됨.
`player_state`를 삭제해도 한타 분석·검색 기능은 영향 없음.

---

### 8-3. 실시간 이벤트 ↔ 온톨로지 연결 패턴

#### 패턴 A: 한타 감지 시 Neo4j 즉시 조회 (RAG 컨텍스트 구성)

```
[한타 감지 (TimescaleDB rule-based + LSTM)]
    ↓
한타 참전 선수 + 챔피언 목록 추출
    ↓
Neo4j 조회 (3가지 병렬):
  ① (Ability)-[:CHAINS_WITH]->(Ability): 현재 조합의 CC 콤보 가능 여부
  ② (Champion)-[:COUNTERS]->(Champion): 구성 카운터 관계·강도
  ③ (Player)-[:MAINS]->(Champion): 각 선수의 해당 챔피언 숙련도
    ↓
"Leona→Orianna 콤보 가능 (windowSec=1.5), 상대는 Jinx 카운터 구성"
→ Bedrock 해설 프롬프트에 주입
```

#### 패턴 B: 과거 한타 임베딩 → Neo4j TeamfightRecord 연결

```
[pgvectorscale: 현재 상태 벡터 → 유사 과거 한타 TOP-20]
    ↓
fight_id 목록 반환
    ↓
Neo4j 조회:
  (TeamfightRecord {id: fight_id})-[:HAD_TEAMFIGHT]-(Game)
  -[:PART_OF]-(Series)-[:COMPETED_IN]-(Team)
  → "유사 상황 20건 중 블루팀 승률 65%, T1 포함 시 72%"
    ↓
팀 특화 한타 예측으로 고도화
```

#### 패턴 C: 경기 종료 후 Neo4j 업데이트 (누적 학습 루프)

```
경기 종료
    ↓
① TeamfightRecord 노드 MERGE → Neo4j
② PARTICIPATED_IN 관계 생성 (선수별 한타 기여)
③ Player.avgFightKP / avgFightGoldSwing 갱신
④ COUNTERS.strength 보정 (실제 프로 경기 데이터 반영)
    ↓
다음 경기의 RAG 품질 자동 향상 (누적될수록 정확도 ↑)
```

---

## 9. Neo4j 운영 구성 (현재) / Neptune 이전 계획

| 항목 | 현재 (Neo4j Desktop) | 상용화 (Neptune) |
|------|---------------------|----------------|
| 인스턴스 | 로컬 bolt://localhost:7687 | db.r6g.large (메모리 16GB) |
| 복제 | 없음 | 1 Primary + 1 Read Replica |
| 백업 | 수동 | 7일 자동 스냅샷 |
| 쿼리 | openCypher (동일) | openCypher |
| 패치 업데이트 | 수동 `python run_pipeline.py --step champions` | EventBridge → Lambda → Neptune |
| 이적 처리 | 수동 `python run_pipeline.py --step neo4j` | 이적 감지 Lambda → MEMBER_OF 타임스탬핑 |
| ML 연동 | 없음 | Neptune Analytics (그래프 ML) |
