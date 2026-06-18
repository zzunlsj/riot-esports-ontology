---
type: research
date: 2026-04-30
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [Live-Stats-API, WebSocket, receiver, ingestion, real-time, pipeline]
---

# Live Stats API WebSocket 수신기 구현

> 실시간 운용의 핵심 컴포넌트. Riot Esports Live Stats API(WebSocket/STOMP)에서
> 1초 단위 경기 데이터를 수신해 TimescaleDB에 적재하고 한타 감지 파이프라인을 트리거.

---

## 0. 구현 상태 (Gate A · 2026-04-30)

| 항목 | 상태 | 비고 |
|-----|------|------|
| `LiveStatsReceiver` mock 모드 구현 | ✅ 완료 | `scripts/realtime/live_stats_receiver.py` |
| `TeamfightDetector` rule-based | ✅ 완료 | Acc 83.4% / 8,711건 감지 |
| `WinProbPredictor` 실시간 추론 연동 | ✅ 완료 | EXP004 XGBoost (.ubj) 로드, EXP003 LightGBM 폴백 |
| mock replay 전체 2,000경기 처리 | ✅ 완료 | team_state 106,566행 적재 |
| 실 Live Stats API WebSocket 연결 | ⚫ Gate B P0 | 자격증명 미확보 |
| STOMP 프레임 파싱 (§3 설계 코드) | ⚫ Gate B | Gate A는 mock_mode로 우회 |
| Bedrock 해설 생성 실 연결 | ⚫ Gate B P0 | mock 텍스트 출력 중 |

> **경로 주의**: 이 문서의 설계 코드는 `src/receiver/`를 기준으로 작성됨.
> Gate A 실제 구현은 `scripts/realtime/live_stats_receiver.py` (단순화된 구조, mock_mode 지원).
> §3~§6은 Gate B 실 연결 시 적용할 목표 설계임.

---

## 1. 프로토콜 개요

```
Live Stats API 연결 정보 (lolesports.aws-usw2-prod.loltmnt03.lolgame.livestats 기준):

rfc190Scope: "lolesports.aws-usw2-prod.loltmnt03.lolgame.livestats"
               └ 지역별 변경 가능: loltmnt01(LPL), loltmnt03(LCK)

프로토콜:  STOMP over WebSocket
URL 패턴:  wss://livestats.proxy.lolesports.com/details?...
           wss://feed.lolesports.com/livestats/v1/window/{gameId}

이벤트 스트림:
  - game_metadata    (경기 시작 시 1회)
  - game_info        (경기 시작 시 1회)
  - champ_select     (챔피언 선택 단계)
  - stats_update     (1초 간격, 전체 플레이어 상태)
  - skill_used       (스킬 사용 시)
  - summoner_spell_used
  - channeling_started / channeling_ended
  - champion_kill    (킬 발생 시, deathRecap 포함)
  - item_purchased / item_sold / item_undo / item_destroyed
  - ward_placed / ward_killed
  - building_destroyed
  - objective_defeated
  - game_ended
```

---

## 2. 수신기 아키텍처

```
┌──────────────────────────────────────────────────────────┐
│                  LiveStatsReceiver                        │
│                                                          │
│  WebSocket  ──→  EventRouter  ──→  HandlerRegistry       │
│     │               │                    │               │
│  STOMP/WS        이벤트 타입          핸들러 목록           │
│                  분기 처리                                 │
│                       │                                  │
│           ┌───────────┼───────────┐                      │
│           ▼           ▼           ▼                      │
│    StatsHandler  EventHandler  FightDetector             │
│    (player_state) (game_events) (한타 감지 트리거)         │
│           │           │           │                      │
│           └───────────┴───────────┘                      │
│                       │                                  │
│               BatchWriter (bulk INSERT)                  │
│                       │                                  │
│               TimescaleDB (player_state, game_events)    │
└──────────────────────────────────────────────────────────┘
           │
           └→ WinProbabilityPredictor (10초마다 호출)
           └→ TeamfightDetector (연속 상태 분석)
           └→ CommentaryTrigger (한타 종료 시 Bedrock 호출)
```

---

## 3. 수신기 구현

### 3-1. 메인 수신기 클래스

```python
# src/receiver/live_stats_receiver.py
"""
Live Stats API WebSocket 수신기.
STOMP 메시지 파싱 → 이벤트 타입별 핸들러 라우팅 → TimescaleDB 적재.
"""
import asyncio
import json
import logging
import time
from datetime import datetime, timezone
from typing import Callable

import websockets
import psycopg2
from psycopg2.extras import execute_batch

logger = logging.getLogger(__name__)


class LiveStatsReceiver:
    """
    Live Stats API 실시간 수신기.

    Usage:
        receiver = LiveStatsReceiver(ws_url, db_conn_str, game_id)
        asyncio.run(receiver.run())
    """

    def __init__(
        self,
        ws_url: str,
        db_conn_str: str,
        game_id: str,
        match_id: str,
        batch_size: int = 30,        # 30행 모아서 bulk INSERT
        flush_interval_sec: float = 2.0,
    ):
        self.ws_url = ws_url
        self.db_conn_str = db_conn_str
        self.game_id = game_id
        self.match_id = match_id
        self.batch_size = batch_size
        self.flush_interval_sec = flush_interval_sec

        # 버퍼 (비동기 적재용)
        self._player_state_buf: list[tuple] = []
        self._game_events_buf: list[tuple] = []

        # 상태 추적
        self._last_flush_ts = time.monotonic()
        self._game_started = False
        self._game_ended = False
        self._participant_meta: dict[int, dict] = {}  # {participantId: {champion, team, ...}}

        # 외부 컴포넌트 (선택적 연결)
        self._on_teamfight_detected: Callable | None = None
        self._on_game_ended: Callable | None = None

    def on_teamfight(self, callback: Callable):
        """한타 감지 시 호출할 콜백 등록"""
        self._on_teamfight_detected = callback
        return self

    def on_game_end(self, callback: Callable):
        """경기 종료 시 호출할 콜백 등록"""
        self._on_game_ended = callback
        return self

    async def run(self):
        """WebSocket 연결 → 수신 루프 실행"""
        logger.info(f"Connecting to {self.ws_url}")
        async with websockets.connect(
            self.ws_url,
            ping_interval=20,
            ping_timeout=60,
            max_size=10 * 1024 * 1024,  # 10MB (stats_update는 클 수 있음)
        ) as ws:
            logger.info(f"Connected. Game ID: {self.game_id}")
            async for raw_message in ws:
                try:
                    await self._handle_message(raw_message)
                except Exception as e:
                    logger.error(f"Handler error: {e}", exc_info=True)

                if self._game_ended:
                    logger.info("Game ended. Flushing buffers and closing.")
                    await self._flush(force=True)
                    break

        logger.info("Receiver stopped.")

    async def _handle_message(self, raw: str):
        """원시 메시지 → 이벤트 파싱 → 라우팅"""
        # STOMP 프레임일 경우 body 추출 (구현에 따라 다름)
        if raw.startswith("{"):
            envelope = json.loads(raw)
        else:
            # STOMP FRAME 파싱 (간략)
            lines = raw.split("\n")
            body_start = next(
                (i + 1 for i, l in enumerate(lines) if l == ""), len(lines)
            )
            body_str = "\n".join(lines[body_start:]).rstrip("\x00")
            if not body_str:
                return
            envelope = json.loads(body_str)

        # rfc461Schema: 이벤트 타입 식별자
        schema = envelope.get("rfc461Schema", "")
        payload = envelope.get("payload", envelope)
        now_utc = datetime.now(timezone.utc)

        if schema == "stats_update" or "participants" in payload:
            await self._handle_stats_update(payload, now_utc)
        elif schema == "game_info":
            self._handle_game_info(payload)
        elif schema == "champion_kill":
            await self._handle_champion_kill(payload, now_utc)
        elif schema in ("skill_used",):
            await self._handle_skill_used(payload, now_utc)
        elif schema in ("channeling_started", "channeling_ended"):
            await self._handle_channeling(schema, payload, now_utc)
        elif schema in ("item_purchased", "item_sold", "item_undo"):
            await self._handle_item_event(schema, payload, now_utc)
        elif schema in ("building_destroyed", "objective_defeated"):
            await self._handle_objective_event(schema, payload, now_utc)
        elif schema == "game_ended":
            self._game_ended = True
            await self._handle_game_ended(payload, now_utc)

        # 배치 플러시 (크기 또는 인터벌 기준)
        elapsed = time.monotonic() - self._last_flush_ts
        if (
            len(self._player_state_buf) >= self.batch_size
            or len(self._game_events_buf) >= self.batch_size * 3
            or elapsed >= self.flush_interval_sec
        ):
            await self._flush()
```

### 3-2. stats_update 핸들러

```python
    async def _handle_stats_update(self, payload: dict, ts: datetime):
        """
        stats_update 이벤트 → player_state 버퍼에 추가.
        참가자 10명 × 1초 = 초당 10행 생성.
        """
        participants = payload.get("participants") or payload.get("allPlayers", [])
        game_time_sec = payload.get("gameTimeOffset", 0) / 1000

        for p in participants:
            pid = p.get("participantId") or p.get("id")
            pos = p.get("position", {})
            team_id = self._participant_meta.get(pid, {}).get("team_id")

            row = (
                ts,                                            # time
                self.match_id,                                 # match_id
                pid,                                           # participant_id
                self._participant_meta.get(pid, {}).get("champion_id"),  # champion_id
                team_id,                                       # team_id (100/200)
                pos.get("x"),                                  # pos_x
                pos.get("z"),                                  # pos_z  ← z축
                p.get("isAlive", True),                        # alive
                p.get("respawnTimer", 0),                      # respawn_timer
                p.get("health"),                               # health
                p.get("healthMax"),                            # health_max
                # health_pct: GENERATED 컬럼이므로 INSERT 제외
                p.get("primaryAbilityResource"),               # primary_resource
                p.get("totalGold"),                            # total_gold
                p.get("currentGold"),                          # current_gold
                p.get("level"),                                # level
                _to_cd(p.get("ability1CooldownRemaining")),   # q_cd
                _to_cd(p.get("ability2CooldownRemaining")),   # w_cd
                _to_cd(p.get("ability3CooldownRemaining")),   # e_cd
                _to_cd(p.get("ultimateCooldownRemaining")),   # r_cd
                _to_cd(p.get("summonerSpell1CooldownRemaining")),  # flash_cd
                _to_cd(p.get("summonerSpell2CooldownRemaining")),  # tp_cd
            )
            self._player_state_buf.append(row)


def _to_cd(val) -> int | None:
    """쿨타임 float(초) → SMALLINT (0.1초 단위 정수)"""
    if val is None:
        return None
    return int(float(val) * 10)
```

### 3-3. champion_kill 핸들러

```python
    async def _handle_champion_kill(self, payload: dict, ts: datetime):
        """
        champion_kill 이벤트 → game_events 버퍼 + 한타 감지 트리거.
        deathRecap JSONB 저장으로 Level 3+ 해설 생성 지원.
        """
        row = (
            ts,
            self.match_id,
            "CHAMPION_KILL",
            int(payload.get("gameTime", 0)),               # timestamp_ms
            payload.get("killerId"),                        # source_participant_id
            payload.get("victimId"),                        # target_participant_id
            payload.get("victimPosition", {}).get("x"),     # pos_x
            payload.get("victimPosition", {}).get("z"),     # pos_z
            None,  # item_id
            None,  # skill_slot
            None,  # ward_type
            None,  # building_type
            None,  # monster_type
            None,  # dragon_type
            None,  # kill_type
            payload.get("assistingParticipants", []),        # assisting_ids
            payload.get("fightDuration"),                   # fight_duration
            json.dumps(payload.get("deathRecap", [])),       # death_recap JSONB
            None,  # max_cooldown
            None,  # charges_remaining
            None,  # is_interrupted
        )
        self._game_events_buf.append(row)

        # 한타 감지 트리거 (TeamfightDetector가 연결된 경우)
        if self._on_teamfight_detected and payload.get("fightDuration", 0) > 2000:
            # 전투 지속시간 2초 이상이면 한타 이벤트로 간주
            asyncio.create_task(
                self._on_teamfight_detected({
                    "trigger": "champion_kill",
                    "timestamp_ms": payload.get("gameTime"),
                    "killer_id": payload.get("killerId"),
                    "victim_id": payload.get("victimId"),
                    "fight_duration": payload.get("fightDuration"),
                    "death_recap": payload.get("deathRecap", []),
                    "position": payload.get("victimPosition", {}),
                })
            )
```

### 3-4. 배치 플러시 (TimescaleDB bulk INSERT)

```python
    async def _flush(self, force: bool = False):
        """
        버퍼 → TimescaleDB bulk INSERT.
        asyncio 루프에서 호출되므로 run_in_executor로 블로킹 IO 분리.
        """
        if not self._player_state_buf and not self._game_events_buf:
            return

        ps_buf = self._player_state_buf[:]
        ge_buf = self._game_events_buf[:]
        self._player_state_buf.clear()
        self._game_events_buf.clear()
        self._last_flush_ts = time.monotonic()

        loop = asyncio.get_event_loop()
        await loop.run_in_executor(None, self._flush_sync, ps_buf, ge_buf)

    def _flush_sync(self, ps_buf: list, ge_buf: list):
        """동기 DB 적재 (별도 스레드에서 실행)"""
        conn = psycopg2.connect(self.db_conn_str)
        try:
            cur = conn.cursor()

            if ps_buf:
                execute_batch(cur, """
                    INSERT INTO player_state (
                        time, match_id, participant_id, champion_id, team_id,
                        pos_x, pos_z, alive, respawn_timer,
                        health, health_max, primary_resource,
                        total_gold, current_gold, level,
                        q_cd, w_cd, e_cd, r_cd, flash_cd, tp_cd
                    ) VALUES (
                        %s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s
                    )
                    ON CONFLICT DO NOTHING
                """, ps_buf, page_size=200)
                logger.debug(f"Flushed {len(ps_buf)} player_state rows")

            if ge_buf:
                execute_batch(cur, """
                    INSERT INTO game_events (
                        time, match_id, event_type, timestamp_ms,
                        source_participant_id, target_participant_id,
                        pos_x, pos_z, item_id, skill_slot,
                        ward_type, building_type, monster_type, dragon_type, kill_type,
                        assisting_ids, fight_duration, death_recap,
                        max_cooldown, charges_remaining, is_interrupted
                    ) VALUES (
                        %s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s
                    )
                """, ge_buf, page_size=100)
                logger.debug(f"Flushed {len(ge_buf)} game_events rows")

            conn.commit()
        except Exception as e:
            conn.rollback()
            logger.error(f"Flush error: {e}", exc_info=True)
            raise
        finally:
            conn.close()
```

---

## 4. 실시간 한타 감지기 (TeamfightDetector)

```python
# src/detector/teamfight_detector.py
"""
Live Stats API 1초 스트림 기반 실시간 한타 감지.
3가지 규칙의 앙상블으로 false positive 감소.
"""
import asyncio
from dataclasses import dataclass, field
from datetime import datetime, timezone
from collections import deque
import math


@dataclass
class PlayerSnapshot:
    participant_id: int
    team_id: int        # 100 or 200
    pos_x: float
    pos_z: float
    alive: bool
    hp_pct: float
    r_ready: bool       # r_cd == 0
    flash_ready: bool   # flash_cd == 0


@dataclass
class TeamfightCandidate:
    start_time: datetime
    trigger: str                   # "proximity" | "kill" | "objective"
    participants: list[int] = field(default_factory=list)
    kill_count: int = 0
    centroid_x: float = 0.0
    centroid_z: float = 0.0
    confirmed: bool = False


class TeamfightDetector:
    """
    RULE 1: proximity — 상대 팀 2인 이상 1500유닛 이내 + 아군 2인 이상 근접
    RULE 2: kill cluster — 30초 내 3킬 이상 발생 (match-v5 fallback과 동일)
    RULE 3: objective proximity — 오브젝트 스폰 90초 이내 + 양팀 500유닛 이내

    확정 기준: RULE 중 1개 이상 트리거 + 10초 지속
    """

    PROXIMITY_RADIUS = 1500          # 유닛
    TEAMFIGHT_MIN_PARTICIPANTS = 4   # 양팀 합산
    OBJECTIVE_RADIUS = 500
    KILL_WINDOW_SEC = 30
    MIN_KILLS_FOR_FIGHT = 3
    CONFIRM_DURATION_SEC = 10

    # 오브젝트 위치 (맵 좌표, 근사값)
    OBJECTIVES = {
        "baron": (5007, 10471),
        "dragon": (9866, 4414),
        "herald": (5007, 10471),     # 초반에는 전령
    }

    def __init__(self, on_fight_start=None, on_fight_end=None):
        self._snapshots: deque[dict] = deque(maxlen=60)   # 최근 60초 스냅샷
        self._kill_times: deque[float] = deque()           # 킬 발생 타임스탬프
        self._current_fight: TeamfightCandidate | None = None
        self._on_fight_start = on_fight_start
        self._on_fight_end = on_fight_end
        self._game_time_sec: float = 0

    def update_stats(self, game_time_sec: float, players: list[PlayerSnapshot]):
        """stats_update 이벤트 처리 (1초마다 호출)"""
        self._game_time_sec = game_time_sec
        snapshot = {
            "time": game_time_sec,
            "players": {p.participant_id: p for p in players},
        }
        self._snapshots.append(snapshot)
        self._check_rules(players, game_time_sec)

    def update_kill(self, game_time_sec: float, killer_id: int, victim_id: int,
                    fight_duration: int):
        """champion_kill 이벤트 처리"""
        self._kill_times.append(game_time_sec)
        # 30초 이전 킬 제거
        while self._kill_times and game_time_sec - self._kill_times[0] > self.KILL_WINDOW_SEC:
            self._kill_times.popleft()

        if self._current_fight:
            self._current_fight.kill_count += 1
            for pid in (killer_id, victim_id):
                if pid not in self._current_fight.participants:
                    self._current_fight.participants.append(pid)

    def _check_rules(self, players: list[PlayerSnapshot], t: float):
        blue = [p for p in players if p.team_id == 100 and p.alive]
        red = [p for p in players if p.team_id == 200 and p.alive]

        rule1 = self._rule_proximity(blue, red)
        rule2 = self._rule_kill_cluster()
        rule3 = self._rule_objective_proximity(blue, red, t)

        triggered = rule1 or rule2 or rule3

        if triggered and self._current_fight is None:
            trigger_name = "proximity" if rule1 else ("kill" if rule2 else "objective")
            self._current_fight = TeamfightCandidate(
                start_time=datetime.now(timezone.utc),
                trigger=trigger_name,
                participants=[p.participant_id for p in blue + red],
            )
            self._update_centroid(players)
            if self._on_fight_start:
                asyncio.create_task(self._on_fight_start(self._current_fight))

        elif not triggered and self._current_fight:
            # 10초 지속 확인 후 종료
            elapsed = (datetime.now(timezone.utc) - self._current_fight.start_time).seconds
            if elapsed >= self.CONFIRM_DURATION_SEC:
                self._current_fight.confirmed = True
                if self._on_fight_end:
                    asyncio.create_task(self._on_fight_end(self._current_fight))
                self._current_fight = None

    def _rule_proximity(self, blue: list, red: list) -> bool:
        """상대 팀 플레이어 간 거리 기반"""
        close_pairs = 0
        for b in blue:
            for r in red:
                dist = math.hypot(b.pos_x - r.pos_x, b.pos_z - r.pos_z)
                if dist < self.PROXIMITY_RADIUS:
                    close_pairs += 1
        return close_pairs >= 2  # 2쌍 이상 근접

    def _rule_kill_cluster(self) -> bool:
        """30초 내 3킬 이상"""
        return len(self._kill_times) >= self.MIN_KILLS_FOR_FIGHT

    def _rule_objective_proximity(self, blue: list, red: list, t: float) -> bool:
        """오브젝트 스폰 90초 이내 + 양팀 모두 근접"""
        obj = self._next_objective(t)
        if obj is None:
            return False
        ox, oz = obj
        blue_near = any(
            math.hypot(p.pos_x - ox, p.pos_z - oz) < self.OBJECTIVE_RADIUS
            for p in blue
        )
        red_near = any(
            math.hypot(p.pos_x - ox, p.pos_z - oz) < self.OBJECTIVE_RADIUS
            for p in red
        )
        return blue_near and red_near

    def _next_objective(self, t: float):
        """현재 시간 기준 90초 이내 스폰될 오브젝트 좌표 반환 (근사)"""
        # 실제 구현에서는 game_events의 objective 타임스탬프를 추적
        # 여기서는 5분(300s) 간격 드래곤 리스폰을 단순 근사
        dragon_spawn = 300 if t < 300 else (
            t + 300 - (t % 300) if t % 300 < 210 else None
        )
        if dragon_spawn and abs(t - dragon_spawn) <= 90:
            return self.OBJECTIVES["dragon"]
        return None

    def _update_centroid(self, players: list[PlayerSnapshot]):
        if not players or not self._current_fight:
            return
        xs = [p.pos_x for p in players if p.alive]
        zs = [p.pos_z for p in players if p.alive]
        if xs:
            self._current_fight.centroid_x = sum(xs) / len(xs)
            self._current_fight.centroid_z = sum(zs) / len(zs)
```

---

## 5. 전체 파이프라인 오케스트레이터

```python
# src/main.py
"""
Live Stats API 수신기 + 한타 감지 + Win Probability + Bedrock 해설 생성
전체 파이프라인 엔트리포인트
"""
import asyncio
import logging
import os
from src.receiver.live_stats_receiver import LiveStatsReceiver
from src.detector.teamfight_detector import TeamfightDetector
from scripts.inference.win_prob_predictor import WinProbabilityPredictor
from scripts.commentary.commentary_generator import generate_commentary
from scripts.commentary.rag_context_builder import build_rag_context
import psycopg2

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

DB_CONN_STR = os.environ["DB_CONN_STR"]
WS_URL = os.environ["LIVE_STATS_WS_URL"]
GAME_ID = os.environ["GAME_ID"]
MATCH_ID = os.environ["MATCH_ID"]
MODEL_PATH = "models/win_prob_v1.lgb"


async def on_fight_start(fight):
    logger.info(f"[한타 시작] trigger={fight.trigger}, "
                f"centroid=({fight.centroid_x:.0f}, {fight.centroid_z:.0f})")


async def on_fight_end(fight):
    logger.info(f"[한타 종료] kills={fight.kill_count}, participants={fight.participants}")

    # RAG 컨텍스트 구성 + Bedrock 해설 생성 (Level 2 기본, Level 3 딜레이 생성)
    conn = psycopg2.connect(DB_CONN_STR)
    try:
        fight_dict = {
            "fight_id": f"{MATCH_ID}_{fight.start_time.timestamp():.0f}",
            "match_id": MATCH_ID,
            "kills_in_fight": fight.kill_count,
            "duration_sec": 15,   # 실제는 종료시간 - 시작시간
            "outcome": "team_a_win",  # 실제는 kill_count 비교로 판단
            "gold_swing": 0,          # TimescaleDB 조회 후 채움
            "objective_converted": None,
            "centroid_x": fight.centroid_x,
            "centroid_z": fight.centroid_z,
            "pre_state": None,
            "start_timestamp_ms": int(fight.start_time.timestamp() * 1000),
            "win_prob_before": None,
            "win_prob_after": None,
            "win_prob_swing": None,
        }
        context = build_rag_context(fight_dict, {}, conn)

        # Level 1 (즉시 생성 — 레이턴시 < 1초)
        l1_commentary = generate_commentary("L1", context, max_tokens=150)
        logger.info(f"[L1 해설] {l1_commentary}")

        # Level 3 (딜레이 생성 — 4초 허용)
        asyncio.create_task(_generate_deep_commentary(context))

    finally:
        conn.close()


async def _generate_deep_commentary(context):
    l3 = generate_commentary("L3", context, max_tokens=600)
    logger.info(f"[L3 해설]\n{l3}")
    # TODO: WebSocket API Gateway 통해 클라이언트로 전송


async def main():
    # 한타 감지기 초기화
    detector = TeamfightDetector(
        on_fight_start=on_fight_start,
        on_fight_end=on_fight_end,
    )

    # Win Probability 예측기 초기화
    wp_predictor = WinProbabilityPredictor(MODEL_PATH)

    # 수신기 초기화 + 콜백 연결
    receiver = LiveStatsReceiver(
        ws_url=WS_URL,
        db_conn_str=DB_CONN_STR,
        game_id=GAME_ID,
        match_id=MATCH_ID,
    ).on_teamfight(
        lambda event: detector.update_kill(
            event["timestamp_ms"] / 1000,
            event["killer_id"],
            event["victim_id"],
            event["fight_duration"],
        )
    )

    await receiver.run()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 6. Docker 서비스 구성

```yaml
# docker-compose.receiver.yml
# 기존 docker-compose.yml (TimescaleDB)에 추가

services:
  live-stats-receiver:
    build:
      context: .
      dockerfile: Dockerfile.receiver
    environment:
      - DB_CONN_STR=postgresql://loluser:lolpass@timescaledb:5432/lol_ontology
      - LIVE_STATS_WS_URL=${LIVE_STATS_WS_URL}
      - GAME_ID=${GAME_ID}
      - MATCH_ID=${MATCH_ID}
      - LOG_LEVEL=INFO
    depends_on:
      - timescaledb
    restart: on-failure
    volumes:
      - ./models:/app/models:ro
    networks:
      - lol-net

# Dockerfile.receiver
# FROM python:3.11-slim
# WORKDIR /app
# COPY requirements.txt .
# RUN pip install -r requirements.txt
# COPY src/ ./src/
# COPY scripts/ ./scripts/
# CMD ["python", "-m", "src.main"]
```

---

## 7. 성능 목표 및 레이턴시 예산

```
이벤트 발생 → 클라이언트 해설 노출 목표 레이턴시:

stats_update 수신              0ms
  └→ DB flush (배치 2초)     +2,000ms
  └→ 한타 감지 트리거          +10ms
  └→ TeamfightDetector 확정   +10,000ms (10초 지속 확인)
  └→ RAG 컨텍스트 조회         +200ms  (pgvectorscale)
  └→ Bedrock L1 해설 생성     +800ms
     ─────────────────────────────────
  L1 총 레이턴시 (한타 확정 후)  ≈ 11초

champion_kill 수신             0ms
  └→ Bedrock L1 (즉시)        +900ms  (kill 기반, fight_duration>2s)
  └→ Bedrock L3 (비동기)      +4,500ms

목표: L1 ≤ 12초, L3 ≤ 15초 (한타 종료 시점 기준)
```

---

## 8. 로컬 테스트 방법

```python
# tests/test_receiver_local.py
"""
저장된 Live Stats API 샘플 JSON으로 수신기 로컬 테스트.
실제 WebSocket 연결 없이 이벤트 핸들러 단위 검증.
"""
import asyncio
import json
from src.receiver.live_stats_receiver import LiveStatsReceiver

# 기존 수집된 샘플 데이터 (T1 vs HLE, gameID 248249)
SAMPLE_EVENTS = [
    {"rfc461Schema": "game_info", "payload": {...}},
    {"rfc461Schema": "stats_update", "payload": {"participants": [...]}},
    {"rfc461Schema": "champion_kill", "payload": {"killerId": 7, "victimId": 3, ...}},
]

async def test_handlers():
    receiver = LiveStatsReceiver(
        ws_url="ws://localhost:9999",  # 더미
        db_conn_str="postgresql://loluser:lolpass@localhost:5432/lol_ontology",
        game_id="test_game",
        match_id="TEST_001",
    )

    from datetime import datetime, timezone
    now = datetime.now(timezone.utc)

    for event in SAMPLE_EVENTS:
        await receiver._handle_message(json.dumps(event))

    await receiver._flush(force=True)
    print("✅ 로컬 테스트 완료")

asyncio.run(test_handlers())
```

---

## 9. 다음 단계

- [x] 로컬 테스트: 기존 수집 샘플 JSON → 핸들러 단위 검증 ✅ (mock replay 2,000경기 완주)
- [x] TeamfightDetector 통합 테스트 ✅ (Acc 83.4%, 8,711건)
- [x] Win Probability 모델 학습 완료 후 실시간 추론 연동 ✅ (EXP004 AUC 0.8353)
- [ ] WebSocket URL 획득 및 인증 방식 확인 — **Gate B P0**
- [ ] §3 STOMP 프레임 파싱 + `src/receiver/` 구조로 전환 — Gate B
- [ ] AWS Lambda 배포 또는 ECS Fargate 컨테이너화 — Gate D

## 10. 관련 파일

- [[live-stats-api-analysis]] — API 필드 완전 목록
- [[local-prototype-setup]] — TimescaleDB DDL (pos_z, health, cooldown)
- [[data-pipeline-and-teamfight-detection]] — match-v5 기반 한타 감지 (학습용)
- [[win-probability-lightgbm]] — Win Probability 모델
- [[bedrock-prompt-templates]] — 해설 생성 템플릿
- [[_index]] — 프로젝트 전체 개요
