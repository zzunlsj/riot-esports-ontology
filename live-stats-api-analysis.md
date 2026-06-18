---
type: research
date: 2026-04-30
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [API, live-stats, discovery, architecture-update, critical]
---

# ⚡ Riot Esports Live Stats API 분석

> **아키텍처 전면 수정 트리거**: 공유된 실제 데이터 샘플을 분석한 결과, 이전에 "공개 API 미제공"으로 분류한 실시간 데이터(위치·HP·쿨타임·골드)가 **실제로 제공되고 있음을 확인**.

> **구현 상태 (Gate A · 2026-04-30)**:
> - `LiveStatsReceiver` 구현 완료 — `on_frame()` → WinProb 추론 → team_state 적재
> - Gate A는 **mock 모드** 운용 (실 Live Stats API 자격증명 미연결)
> - 실경기 Live Stats API 연결 검증은 **Gate B P0** 항목
> - 이 문서의 스키마(stats_update, champion_kill, deathRecap 등)는 실 연결 시 그대로 활용

---

## 데이터 소스 식별

```json
"rfc190Scope": "lolesports.aws-usw2-prod.loltmnt03.lolgame.livestats"
"platformID": "LOLTMNT03"
"statsUpdateInterval": 1000
```

이는 Riot의 **공식 프로 경기 Live Stats 피드**입니다.

- `LOLTMNT03` = LCK/LCK CL 전용 프로덕션 서버 (Tournament Match Node)
- `rfc190Scope`: RFC-190 기반 내부 라우팅 스코프 → AWS us-west-2 프로덕션 환경
- `statsUpdateInterval: 1000` = **1초 단위 업데이트** ✅
- Riot Esports 공식 데이터 파트너(OGS, GRID 등)를 통해 접근 가능

---

## 이벤트 스키마 전체 목록 (`rfc461Schema`)

### 경기 메타 이벤트

| `rfc461Schema` | 설명 | 핵심 필드 |
|----------------|------|---------|
| `champ_select` | 챔피언 선택 단계 | `teamOne[]`, `teamTwo[]`, `bannedChampions[]`, `gameState` |
| `game_info` | 경기 시작 시 전체 참가자 정보 | `participants[]` (10명 전체 스냅샷) |
| `pause_ended` | 일시정지 해제 | `wallTime` |
| `reconnect` | 재접속 | `participant` |

### **실시간 상태 이벤트 (핵심)**

| `rfc461Schema` | 설명 | 핵심 필드 |
|----------------|------|---------|
| `stats_update` | **1초 단위 전체 상태 스냅샷** | 참가자 10명 전원의 position, health, gold, cooldown 등 |
| `queued_epic_monster_info` | 에픽 몬스터 예고 | `monsterName`, `spawnTime`, `position` |
| `queued_dragon_info` | 드래곤 예고 | `nextDragonName`, `nextDragonSpawnTime` |

### 이벤트 기반 데이터

| `rfc461Schema` | 설명 | 핵심 필드 |
|----------------|------|---------|
| `champion_kill` | 킬 이벤트 (★ 최고 가치) | `killer`, `victim`, `assistants[]`, `position`, `fightDuration`, `deathRecap[]`, `bounty`, `shutdownBounty`, `victimTeamID`, `killerTeamID` |
| `champion_kill_special` | 퍼스트블러드·멀티킬 | `killer`, `killType`, `position` |
| `skill_used` | 스킬 사용 | `participant`, `skillSlot`, `maxCooldown`, `chargesRemaining`, `maxCharges` |
| `skill_level_up` | 스킬 레벨업 | `participant`, `skillSlot`, `evolved` |
| `summoner_spell_used` | 소환사 주문 사용 | `participantID`, `summonerSpellName`, `maxCooldown`, `chargesRemaining` |
| `channeling_started` | 채널링 시작 | `participantID`, `skillSlot`, `channelingType` |
| `channeling_ended` | 채널링 종료 | `participantID`, `skillSlot`, `isInterrupted` |
| `item_purchased` | 아이템 구매 | `participantID`, `itemID`, `gameTime` |
| `item_sold` | 아이템 판매 | `participantID`, `itemID`, `gameTime` |
| `item_undo` | 아이템 취소 | `participantID`, `itemID`, `itemBeforeUndo`, `goldGain` |
| `item_destroyed` | 아이템 소멸 | `participantID`, `itemID` |
| `item_active_ability_used` | 아이템 액티브 사용 | `participantID`, `itemID`, `inventorySlot`, `maxCooldown` |
| `item_modified` | 아이템 변환 (Ornn 등) | `participantID`, `itemID`, `itemModifierId` |
| `champion_level_up` | 레벨업 | `participant`, `level` |
| `epic_monster_kill` | 에픽 몬스터 처치 | `killer`, `monsterType`, `killType`, `killerGold`, `killerTeamID`, `position` |
| `epic_monster_spawn` | 에픽 몬스터 스폰 | `monsterType`, `dragonType`, `position` |
| `building_destroyed` | 건물 파괴 | `buildingType`, `turretTier`, `teamID`, `lane`, `lastHitter`, `position` |
| `turret_plate_destroyed` | 포탑 플레이트 파괴 | `teamID`, `lane`, `position`, `lastHitter` |
| `turret_plate_gold_earned` | 포탑 플레이트 골드 | `participantID`, `teamID`, `bounty` |
| `building_gold_grant` | 건물 골드 지급 | `teamID`, `lane`, `amount`, `recipientId`, `source` |
| `ward_placed` | 와드 설치 | `wardType`, `placer`, `position` |
| `ward_killed` | 와드 파괴 | `wardType`, `killer`, `position` |
| `neutral_minion_spawn` | 중립 미니언 스폰 | `monsterType`, `teamSide`, `position` |
| `feat_updated` | 특수 업적 업데이트 | `teamID`, `featType`, `stacks` |
| `objective_bounty_prestart` | 오브젝트 바운티 예고 | `teamID`, `actualStartTime` |

---

## `stats_update` 이벤트 — 참가자 필드 전체

> 1초 단위로 10명 전원의 상태가 도착. 이전 가정이었던 "실시간 데이터 불가"를 **완전 번복**함.

```json
{
  "participantID": 1,
  "championName": "Yorick",
  "teamID": 100,
  "playerName": "T1 Doran",
  "puuid": "...",
  "riotId": { "displayName": "T1 Doran", "tagLine": "eProd" },
  "skinID": 2,
  "alive": true,
  "respawnTimer": 0,
  "level": 1,
  "XP": 0,
  "XPForNextLevel": 280,

  // ── 위치 (좌표계: x, z) ──────────────────────
  "position": { "x": 554, "z": 581 },    // ← pos_z (y 아님!)

  // ── HP ──────────────────────────────────────
  "health": 650,
  "healthMax": 650,
  "healthRegen": 0,

  // ── 마나/리소스 ────────────────────────────
  "primaryAbilityResource": 300,
  "primaryAbilityResourceMax": 300,
  "primaryAbilityResourceRegen": 0,

  // ── 골드 ────────────────────────────────────
  "currentGold": 500,
  "totalGold": 500,
  "goldPerSecond": 0,
  "shutdownValue": 0,
  "goldStats": { "starting": 500 },

  // ── 스킬 쿨타임 (초 단위) ─────────────────
  "ability1CooldownRemaining": 0,
  "ability2CooldownRemaining": 0,
  "ability3CooldownRemaining": 0,
  "ability4CooldownRemaining": 0,
  "ultimateCooldownRemaining": 0,
  "ability1Level": 0,
  "ability2Level": 0,
  "ability3Level": 0,
  "ability4Level": 0,
  "ability1Name": "YorickQ",
  "ability2Name": "YorickW",
  "ability3Name": "YorickE",
  "ability4Name": "YorickR",
  "ultimateName": "YorickR",

  // ── 소환사 주문 쿨타임 ─────────────────────
  "summonerSpell1Name": "SummonerTeleport",
  "summonerSpell2Name": "SummonerFlash",
  "summonerSpell1CooldownRemaining": 14.886,
  "summonerSpell2CooldownRemaining": 14.886,

  // ── 전투 스탯 ──────────────────────────────
  "attackDamage": 25,
  "abilityPower": 0,
  "armor": 39,
  "magicResist": 32,
  "attackSpeed": 100,
  "lifeSteal": 0,
  "spellVamp": 0,
  "cooldownReduction": 0,
  "armorPenetration": 0,
  "armorPenetrationPercent": 0,
  "armorPenetrationPercentBonus": 0,
  "magicPenetration": 0,
  "magicPenetrationPercent": 0,
  "magicPenetrationPercentBonus": 0,
  "ccReduction": 0,

  // ── 아이템 목록 ─────────────────────────────
  "items": [
    { "itemID": 3599, "itemCooldown": 0 }
  ],

  // ── 버프 ──────────────────────────────────
  "stackingBuffs": [
    { "stacks": 8, "id": 268294393 }
  ],

  // ── 상세 스탯 배열 ──────────────────────────
  "stats": [
    { "name": "MINIONS_KILLED", "value": 0 },
    { "name": "NEUTRAL_MINIONS_KILLED", "value": 0 },
    { "name": "CHAMPIONS_KILLED", "value": 0 },
    { "name": "NUM_DEATHS", "value": 0 },
    { "name": "ASSISTS", "value": 0 },
    { "name": "WARD_PLACED", "value": 0 },
    { "name": "WARD_KILLED", "value": 0 },
    { "name": "VISION_SCORE", "value": 0 },
    { "name": "TOTAL_DAMAGE_DEALT_TO_CHAMPIONS", "value": 0 },
    { "name": "PHYSICAL_DAMAGE_DEALT_TO_CHAMPIONS", "value": 0 },
    { "name": "MAGIC_DAMAGE_DEALT_TO_CHAMPIONS", "value": 0 },
    { "name": "TOTAL_DAMAGE_TAKEN", "value": 0 },
    { "name": "TOTAL_TIME_CROWD_CONTROL_DEALT_TO_CHAMPIONS", "value": 0 },
    { "name": "TOTAL_HEAL_ON_TEAMMATES", "value": 0 },
    ...
  ]
}
```

### `stats_update` 팀 필드

```json
{
  "teamID": 100,
  "totalGold": 2500,
  "towerKills": 0,
  "championsKills": 0,
  "deaths": 0,
  "assists": 0,
  "dragonKills": 0,
  "baronKills": 0,
  "inhibKills": 0
}
```

---

## `champion_kill` 이벤트 — 특별 필드

```json
{
  "killer": 4,
  "victim": 9,
  "assistants": [2, 5],
  "position": { "x": 3684, "z": 13997 },
  "fightDuration": 2.605,         // ← 전투 지속 시간 (초)
  "bounty": 300,
  "shutdownBounty": 0,
  "killStreakLength": 0,
  "victimTeamID": 200,
  "killerTeamID": 100,
  "gameTime": 523924,             // ms 단위
  "deathRecap": [                 // ← 피해 상세 분석 (스킬별)
    {
      "casterId": 4,
      "source": "Champion",
      "breakdown": [
        {
          "datadragonID": "MissFortunePassive",
          "isAutoAttack": false,
          "physicalDmg": 120, "magicDmg": 0, "trueDmg": 0
        },
        ...
      ]
    }
  ]
}
```

`deathRecap`은 **해설 생성에 직접 활용 가능**: "이 킬에서 MF가 W→Q 순으로 238 물리 피해를 입혔고, Pantheon이 추가로 362 피해..."

---

## 아키텍처 수정 — 이전 vs 현재

### TRACK 재정의

| 구분 | 이전 | 수정 후 |
|------|------|---------|
| TRACK A | match-v5 timeline (1분 간격, 과거) | **Live Stats API (1초 간격, 실시간)** |
| TRACK B | GRID 파트너십 (B2B) | 동일 (대규모 상용화 시) |

### player_state 테이블 수정 (1초 → 좌표 z축, 추가 컬럼)

```sql
CREATE TABLE player_state (
  time                    TIMESTAMPTZ   NOT NULL,
  match_id                TEXT          NOT NULL,
  participant_id          SMALLINT      NOT NULL,
  champion_name           TEXT,
  team_id                 SMALLINT,

  -- 위치 (z축, y축 아님!)
  pos_x                   INT,
  pos_z                   INT,          -- ← 수정: z축

  -- 상태
  health                  FLOAT,
  health_max              FLOAT,
  health_pct              FLOAT GENERATED ALWAYS AS (health / NULLIF(health_max,0)) STORED,
  primary_resource        FLOAT,
  primary_resource_max    FLOAT,
  alive                   BOOLEAN,
  respawn_timer           FLOAT,

  -- 골드
  current_gold            INT,
  total_gold              INT,
  shutdown_value          INT,

  -- 레벨
  level                   SMALLINT,
  xp                      INT,

  -- 스킬 쿨타임 (초)
  q_cd                    FLOAT,
  w_cd                    FLOAT,
  e_cd                    FLOAT,
  r_cd                    FLOAT,
  flash_cd                FLOAT,        -- summonerSpell1 or 2 if flash
  tp_cd                   FLOAT,

  -- 아이템 (JSON)
  items                   JSONB,

  -- 버프 (JSON)
  stacking_buffs          JSONB,

  -- 임베딩
  state_embedding         vector(128),

  PRIMARY KEY (time, match_id, participant_id)
);

SELECT create_hypertable('player_state', by_range('time'));
```

### game_events 테이블 수정 — 추가 이벤트 타입

```sql
-- 기존 event_type 목록에 추가
-- 'skill_used', 'summoner_spell_used', 'channeling_started', 'channeling_ended'
-- 'epic_monster_spawn', 'epic_monster_kill', 'neutral_minion_spawn'
-- 'turret_plate_destroyed', 'building_gold_grant'
-- 'feat_updated', 'objective_bounty_prestart'
-- 'ward_placed', 'ward_killed'

ALTER TABLE game_events
  ADD COLUMN max_cooldown      INT,        -- skill_used 쿨타임
  ADD COLUMN charges_remaining SMALLINT,
  ADD COLUMN is_interrupted    BOOLEAN,    -- channeling_ended
  ADD COLUMN fight_duration    FLOAT,      -- champion_kill
  ADD COLUMN death_recap       JSONB,      -- champion_kill 상세
  ADD COLUMN shutdown_bounty   INT,
  ADD COLUMN in_enemy_jungle   BOOLEAN,
  ADD COLUMN dragon_type       TEXT,       -- epic_monster_spawn
  ADD COLUMN feat_type         TEXT,       -- feat_updated
  ADD COLUMN stacks            SMALLINT;   -- feat_updated
```

---

## 수집 파이프라인 수정

```
[Live Stats API]  ←  Kinesis Data Streams  (eventbridge webhook 또는 polling)
    │
    ├── stats_update (1초) → player_state + team_state
    ├── champion_kill → game_events + teamfight_events (trigger)
    ├── skill_used / summoner_spell_used → game_events
    ├── item_purchased/sold/modified → game_events
    ├── epic_monster_kill → game_events
    ├── building_destroyed → game_events
    └── ward_placed/killed → game_events
```

---

## 한타 감지 전략 업그레이드

이제 실시간 위치 데이터가 있으므로 **원래 설계한 집결 감지 방식 복원 가능**.

```python
# 한타 감지 (업그레이드)
# 두 가지 트리거를 앙상블:
#
# RULE 1 (실시간 집결): stats_update에서
#   양팀 2인 이상이 1500유닛 이내 → 30초 내 추가 접근
#
# RULE 2 (이벤트 기반): champion_kill이 30초/2000유닛 내 클러스터
#
# RULE 3 (오브젝트): queued_epic_monster_info 스폰 90초 이내 양팀 집결
#
# ML: LSTM P(teamfight) > 0.75
#
# → RULE 1~3 중 2개 이상 AND ML > 0.5 → 한타 예측 발동
```

`skill_used` + `summoner_spell_used` + `channeling_started`를 통해 **스킬 사용 패턴 실시간 추적**도 가능 → 진입기(initiator) 스킬 감지 → 한타 예측 강화.

---

## deathRecap 활용 — 해설 품질 향상

```python
# Bedrock 해설 입력 구조 업그레이드
kill_context = {
    "killer": "T1 Gumayusi (MissFortune)",
    "victim": "HLE Viper (Kalista)",
    "assistants": ["T1 Oner (Pantheon)", "T1 Keria (Neeko)"],
    "fight_duration_sec": 2.6,
    "death_recap": [
        {"caster": "MissFortune", "skills": ["Passive", "Q", "AA"], "total_dmg": 383},
        {"caster": "Pantheon", "skills": ["W", "Q", "E2"], "total_dmg": 362},
        {"caster": "Neeko", "skills": ["R", "AA", "ItemGhostWard", "E", "Q"], "total_dmg": 382}
    ],
    "kill_type": "firstBlood"
}

# Level 3+ 해설 가능 예시:
# "Keria의 Neeko R로 선제 CC를 건 후, Oner의 Pantheon W+Q 콤보(362 물리)와
#  Gumayusi의 Passive+Q 조합(383 물리)이 동시에 터지며 2.6초 만에 처치.
#  첫 피가 T1으로 넘어가며 조기 심리적 우위 확보."
```

---

## 결론

보내준 데이터 샘플로 확인된 내용:

- **실시간 위치(1초)**  → ✅ 제공
- **실시간 HP/마나**    → ✅ 제공
- **스킬 쿨타임**       → ✅ 제공 (stats_update + skill_used 이벤트)
- **소환사 주문 쿨타임** → ✅ 제공
- **실시간 골드**       → ✅ 제공
- **데스리캡(스킬별 피해)** → ✅ 제공
- **채널링 인터럽트 감지** → ✅ 제공

**당초 설계한 아키텍처가 그대로 유효**하며, player_state 테이블의 z축 수정과 좌표계 반영만 필요.
공개 API 대신 이 Live Stats API를 직접 소비하는 구조로 전환.

---

## 관련 파일

- [[api-verification]] — TRACK A/B 전략 (2026-04-30 업데이트 완료)
- [[local-prototype-setup]] — DDL (pos_z, health, cooldown 컬럼 반영 완료)
- [[_index]] — 아키텍처 전체 개요
- [[architecture-mermaid]] — 현재 구현 아키텍처 (Gate A 기준)
