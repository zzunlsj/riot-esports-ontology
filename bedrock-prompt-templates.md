---
type: research
date: 2026-04-30
project: Riot Esports 실시간 경기 맥락 온톨로지
status: active
tags: [Bedrock, Claude, prompt, template, commentary, level1, level2, level3, level4, RAG]
---

# Bedrock 프롬프트 수준별 템플릿 설계

> 유저 수준별 딥다이브 해설 생성을 위한 Amazon Bedrock (Claude) 프롬프트 시스템 설계.
> Live Stats API deathRecap + pgvectorscale 유사 상황 검색 → RAG 기반 해설.

---

## 0. 현재 구현 상태 (Gate A · 2026-04-30)

| 컴포넌트 | 상태 | 비고 |
|---------|------|------|
| 해설 수준 정의 (L1~L4) | ✅ 설계 완료 | §1 참조 |
| 프롬프트 템플릿 (L1~L4) | ✅ 설계 완료 | §3 참조 |
| commentary_generator.py | 🟡 mock 모드 | AWS 크레덴셜 미연결 (NoCredentialsError) |
| RAG rag_context_builder.py | 🟡 부분 구현 | pgvectorscale game_state K-NN 미구현 (§2 참조) |
| deathRecap formatter | ✅ 구현 완료 | §5 참조 |
| Bedrock 실 호출 | ⚫ Gate B | AWS 크레덴셜 연결 후 활성화 예정 |

> **Gate B 완료 후**: NoCredentialsError 해소, E2E 레이턴시 측정, 성능 목표 실측 (§8)

---

## 1. 해설 수준 정의

| 레벨 | 대상 유저 | 핵심 컨텐츠 | 예상 길이 |
|------|---------|-----------|---------|
| Level 1 | 라이트/관전 입문 | 결과 요약 + 쉬운 설명 | 2-3문장 |
| Level 2 | 일반 시청자 | 핵심 플레이 + 게임 영향 | 5-7문장 |
| Level 3 | 하드코어 팬 | 스킬 분석 + 타이밍 판단 | 10-15문장 |
| Level 4 | 분석가/프로 지망 | 전략적 맥락 + 대안 시나리오 | 구조화 보고서 |

---

## 2. RAG 컨텍스트 구성

> **구현 상태 (Gate A)**:
> - `teamfight_events.pre_state` 벡터 컬럼: ⚫ 미구현 — K-NN 유사 한타 검색 비활성
> - `composition_embeddings` (PCA 64dim): ✅ 1,572 nodes 적재 완료, K-NN 연동은 Gate B
> - Neo4j 정적 온톨로지 (챔피언·스킬): ✅ 구현 완료 (아래 §9에서 neptune-schema → Neo4j)
> - 현재 유사 한타 검색은 rule-based fallback 또는 빈 결과 반환

```python
# scripts/commentary/rag_context_builder.py
"""
한타 이벤트 발생 시 Bedrock 프롬프트용 RAG 컨텍스트 구성.
pgvectorscale → 유사 과거 한타 검색 (Gate B 이후 활성화).
Neo4j → 챔피언 스킬/시너지 정보 (구현 완료).
"""
import json
import psycopg2
import numpy as np


def build_rag_context(
    fight: dict,       # teamfight_events 행
    match_meta: dict,  # spectator-v5 메타 (챔피언, 선수)
    conn              # psycopg2 connection
) -> dict:
    """
    한타 이벤트 → Bedrock 프롬프트 컨텍스트 딕셔너리 반환
    """
    cur = conn.cursor()

    # ① 유사 과거 한타 5건 검색 (pgvectorscale — Gate B 이후 활성화)
    # NOTE: teamfight_events.pre_state 벡터 컬럼 미구현 → similar_fights=[] 반환
    cur.execute("""
        SELECT
            te.fight_id,
            te.match_id,
            te.outcome,
            te.gold_swing,
            te.kills_in_fight,
            te.objective_converted,
            te.pre_state <=> %s::vector AS distance
        FROM teamfight_events te
        WHERE te.fight_id != %s
        ORDER BY distance ASC
        LIMIT 5
    """, (str(fight["pre_state"]), fight["fight_id"]))
    similar_fights = cur.fetchall()

    # ② deathRecap 파싱 (champion_kill 이벤트에서 추출)
    cur.execute("""
        SELECT
            death_recap,
            source_participant_id AS killer_id,
            target_participant_id AS victim_id,
            fight_duration
        FROM game_events
        WHERE match_id = %s
          AND event_type = 'CHAMPION_KILL'
          AND timestamp_ms BETWEEN %s AND %s
        ORDER BY timestamp_ms
    """, (
        fight["match_id"],
        fight["start_timestamp_ms"],
        fight["start_timestamp_ms"] + fight["duration_sec"] * 1000
    ))
    kill_events = cur.fetchall()

    # ③ 한타 전 gold_swing 및 Win Probability 변화
    wp_swing = abs(
        (fight.get("win_prob_after") or 0.5) -
        (fight.get("win_prob_before") or 0.5)
    )

    # ④ 참전 챔피언 메타 (match_meta에서)
    participants_meta = []
    for pid in fight.get("participants", {}).get("team_a", []) + \
               fight.get("participants", {}).get("team_b", []):
        champ = match_meta.get(str(pid), {})
        participants_meta.append({
            "participant_id": pid,
            "champion": champ.get("championName", "Unknown"),
            "player": champ.get("riotId", "Unknown"),
            "team": "blue" if pid <= 5 else "red",
        })

    return {
        "fight_summary": {
            "kills": fight["kills_in_fight"],
            "duration_sec": fight["duration_sec"],
            "outcome": fight["outcome"],
            "gold_swing": fight["gold_swing"],
            "objective_converted": fight["objective_converted"],
            "win_prob_before": fight.get("win_prob_before"),
            "win_prob_after": fight.get("win_prob_after"),
            "wp_swing": round(wp_swing, 3),
            "location": {
                "centroid_x": fight["centroid_x"],
                "centroid_z": fight["centroid_z"],
            }
        },
        "kill_details": [
            {
                "killer": k[1], "victim": k[2],
                "fight_duration_ms": k[3],
                "death_recap": k[0]  # JSONB: [{spellName, damage, ...}]
            }
            for k in kill_events
        ],
        "participants": participants_meta,
        "similar_fights": [
            {
                "outcome": sf[2],
                "gold_swing": sf[3],
                "kills": sf[4],
                "objective": sf[5],
                "similarity": round(1 - sf[6], 3)
            }
            for sf in similar_fights
        ],
    }
```

---

## 3. 수준별 시스템 프롬프트

### Level 1 — 라이트/관전 입문

```python
SYSTEM_PROMPT_L1 = """
당신은 리그 오브 레전드 관전 해설가입니다.
처음 LoL을 보는 시청자에게 이야기하듯 친근하고 쉽게 설명하세요.

규칙:
- 게임 용어는 반드시 괄호로 설명 (예: 한타(집단 싸움), 바론(강력한 몬스터))
- 결과와 승패 영향에만 집중
- 최대 2-3문장으로 요약
- 긍정적이고 흥미롭게 표현
"""

USER_TEMPLATE_L1 = """
아래 한타(집단 싸움) 정보를 보고 간단히 설명해주세요.

결과: {outcome}
주요 킬 수: {kills}명
시간: {duration_sec}초 동안 싸움
골드 변화: {gold_swing:+,}골드 차이 발생
오브젝트: {objective_converted}

예시 형식:
"T1이 집단 싸움에서 4명을 잡아내며 큰 승리를 거뒀습니다! 이어서 바론(강력한 버프 몬스터)까지 획득하며 게임의 흐름이 크게 바뀌었습니다."
"""
```

### Level 2 — 일반 시청자

```python
SYSTEM_PROMPT_L2 = """
당신은 리그 오브 레전드 전문 해설가입니다.
LoL을 어느 정도 아는 시청자를 대상으로 해설합니다.

규칙:
- 핵심 스킬 콤보와 킬 순서 설명
- 골드 스윙과 오브젝트 전환의 게임 영향 설명
- 승률 변화가 있다면 언급
- 5-7문장 내외
- 비교 과거 사례가 있다면 간략히 언급
"""

USER_TEMPLATE_L2 = """
다음 한타 데이터를 바탕으로 해설을 작성하세요.

[한타 요약]
- 결과: {outcome}
- 킬 수: {kills}
- 지속 시간: {duration_sec}초
- 골드 변화: {gold_swing:+,}골드
- 오브젝트 획득: {objective_converted}
- 승률 변화: {win_prob_before:.0%} → {win_prob_after:.0%} ({wp_swing:+.0%})

[주요 킬 순서]
{kill_sequence}

[참전 챔피언]
블루팀: {blue_champions}
레드팀: {red_champions}

유사 과거 한타 패턴 (참고용): {similar_fight_pattern}
"""
```

### Level 3 — 하드코어 팬

```python
SYSTEM_PROMPT_L3 = """
당신은 리그 오브 레전드 심층 분석 전문가입니다.
경쟁적으로 LoL을 즐기거나 세부 분석에 관심 있는 팬을 위해 해설합니다.

분석 포인트:
1. 진입 타이밍과 플래시/궁 준비 상태
2. 스킬 콤보 연계 (CC 체인, 딜 순서)
3. 포지셔닝의 적절성
4. 게임체인저 플레이 식별
5. 놓친 기회 또는 실수 분석
6. 대안 시나리오 제시

형식: 자연스러운 문단, 10-15문장
"""

USER_TEMPLATE_L3 = """
다음 한타의 심층 분석을 작성하세요.

[한타 컨텍스트]
- 게임 시간: {game_time}
- 위치: {location_zone}
- 발생 이유: {trigger} (오브젝트 {objective_nearby} 스폰 {seconds_to_obj}초 전)
- 승률: {win_prob_before:.1%} → {win_prob_after:.1%} (변화량 {wp_swing:+.1%})

[구성 분석]
블루팀: {blue_comp_analysis}
  - 구성 타입: {blue_comp_type}
  - engage_potential: {blue_engage}
레드팀: {red_comp_analysis}
  - 구성 타입: {red_comp_type}
  - engage_potential: {red_engage}

[스킬 사용 순서]
{skill_usage_sequence}

[킬 분석 (deathRecap)]
{death_recap_analysis}

[유사 과거 사례]
{similar_cases}

다음을 반드시 포함하세요:
1. 진입 타이밍 평가 (적절/조금 이른/늦은 이유)
2. 핵심 스킬 콤보 설명
3. 게임체인저 플레이 1건 강조
4. "만약 ~했다면" 대안 시나리오 1가지
"""
```

### Level 4 — 분석가/프로 지망

```python
SYSTEM_PROMPT_L4 = """
당신은 LCK/LPL 수준의 경기를 분석하는 수석 전략 분석가입니다.
프로팀 코칭스태프, 데이터 분석가, 유망 선수를 대상으로 보고서를 작성합니다.

보고서 구조 (반드시 준수):
## 1. 한타 요약 (Executive Summary)
## 2. 사전 세팅 분석 (Pre-fight Setup)
## 3. 실행 분석 (Execution Analysis)
## 4. 수치 임팩트 (Quantitative Impact)
## 5. 전략적 의미 (Strategic Implication)
## 6. 개선 포인트 (Coaching Notes)
## 7. 유사 프로 경기 사례 (Historical Benchmark)

각 섹션은 구체적 수치와 타임스탬프를 포함해야 합니다.
"""

USER_TEMPLATE_L4 = """
다음 데이터를 기반으로 전문 분석 보고서를 작성하세요.

[경기 정보]
- 매치 ID: {match_id}
- 게임 시간: {game_time} ({game_phase} phase)
- 한타 ID: {fight_id}

[사전 상태]
- 골드차: {gold_diff:+,}골드 ({gold_adv_team} 우세)
- 타워: 블루 {towers_blue} - 레드 {towers_red}
- 드래곤: 블루 {dragons_blue} - 레드 {dragons_red}
- 다음 오브젝트: {next_objective} ({seconds_to_obj}초 후 스폰)
- 시야 상태: {vision_state}

[구성 상세]
{composition_full}

[스킬 사용 로그]
{skill_log}

[킬 상세 (deathRecap 포함)]
{death_recap_full}

[Win Probability 추이]
- 한타 5분 전: {wp_5min_before:.1%}
- 한타 직전: {wp_before:.1%}
- 한타 직후: {wp_after:.1%}
- 스윙 크기: {wp_swing:+.1%}

[유사 과거 한타 (pgvectorscale TOP-3)]
{similar_fights_detailed}

보고서를 작성하세요.
"""
```

---

## 4. 해설 생성 오케스트레이터

```python
# scripts/commentary/commentary_generator.py
"""
한타 이벤트 → 수준별 해설 생성 파이프라인
Amazon Bedrock Claude API 호출

현재 상태 (Gate A): AWS 크레덴셜 미연결 → mock 모드 운용 중.
  NoCredentialsError 발생 시 LiveStatsReceiver가 mock fallback 텍스트 반환.
Gate B: 실 크레덴셜 연결 후 아래 MODEL_ID 활성화.
"""
import boto3
import json
from typing import Literal

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
# Claude 4 Sonnet (2026-04 기준 최신) — 정확한 Bedrock ARN은 콘솔에서 확인
MODEL_ID = "anthropic.claude-sonnet-4-6-20250514-v1:0"

LevelType = Literal["L1", "L2", "L3", "L4"]

SYSTEM_PROMPTS = {
    "L1": SYSTEM_PROMPT_L1,
    "L2": SYSTEM_PROMPT_L2,
    "L3": SYSTEM_PROMPT_L3,
    "L4": SYSTEM_PROMPT_L4,
}


def build_user_prompt(level: LevelType, context: dict) -> str:
    """RAG 컨텍스트 → 레벨별 user 프롬프트 조립"""
    fs = context["fight_summary"]
    participants = context["participants"]

    blue_champs = [p["champion"] for p in participants if p["team"] == "blue"]
    red_champs = [p["champion"] for p in participants if p["team"] == "red"]

    # kill_details → 텍스트 변환
    kill_seq_lines = []
    for k in context["kill_details"]:
        recap = k.get("death_recap") or []
        top_damage = sorted(recap, key=lambda x: x.get("totalDamage", 0), reverse=True)[:2]
        recap_str = ", ".join(
            f"{d.get('spellName', '?')} {d.get('totalDamage', 0):,}딜"
            for d in top_damage
        )
        kill_seq_lines.append(
            f"- 참가자{k['killer_id']} → 참가자{k['victim_id']} ({recap_str})"
        )
    kill_seq = "\n".join(kill_seq_lines) if kill_seq_lines else "정보 없음"

    similar_pattern = ""
    if context["similar_fights"]:
        sf = context["similar_fights"][0]
        similar_pattern = (
            f"유사도 {sf['similarity']:.0%} — {sf['outcome']}, "
            f"킬 {sf['kills']}건, 골드 {sf['gold_swing']:+,}"
        )

    if level == "L1":
        return USER_TEMPLATE_L1.format(
            outcome="블루팀 승리" if "team_a_win" in fs["outcome"] else "레드팀 승리",
            kills=fs["kills"],
            duration_sec=fs["duration_sec"],
            gold_swing=fs["gold_swing"],
            objective_converted=fs["objective_converted"] or "없음",
        )
    elif level == "L2":
        return USER_TEMPLATE_L2.format(
            outcome=fs["outcome"],
            kills=fs["kills"],
            duration_sec=fs["duration_sec"],
            gold_swing=fs["gold_swing"],
            objective_converted=fs["objective_converted"] or "없음",
            win_prob_before=fs.get("win_prob_before") or 0.5,
            win_prob_after=fs.get("win_prob_after") or 0.5,
            wp_swing=fs.get("wp_swing") or 0.0,
            kill_sequence=kill_seq,
            blue_champions=", ".join(blue_champs),
            red_champions=", ".join(red_champs),
            similar_fight_pattern=similar_pattern or "유사 사례 없음",
        )
    # L3, L4는 더 많은 컨텍스트 필요 (별도 구현)
    else:
        raise NotImplementedError(f"Level {level} prompt not fully implemented")


def generate_commentary(
    level: LevelType,
    context: dict,
    max_tokens: int = 800,
) -> str:
    """Bedrock Claude API 호출 → 해설 텍스트 반환"""
    system_prompt = SYSTEM_PROMPTS[level]
    user_prompt = build_user_prompt(level, context)

    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": max_tokens,
        "system": system_prompt,
        "messages": [{"role": "user", "content": user_prompt}],
    }

    response = bedrock.invoke_model(
        modelId=MODEL_ID,
        body=json.dumps(body),
        contentType="application/json",
    )

    result = json.loads(response["body"].read())
    return result["content"][0]["text"]
```

---

## 5. deathRecap → 해설 변환 유틸리티

```python
# scripts/commentary/death_recap_formatter.py
"""
champion_kill.deathRecap[] → 인간이 읽기 좋은 해설 텍스트로 변환.
Level 3+ 해설 품질의 핵심.
"""

def format_death_recap(death_recap: list, champion_map: dict) -> str:
    """
    death_recap: [
        {
            "spellName": "Orianna_R",
            "spellIconPath": "...",
            "displayName": "Command: Shockwave",
            "rawDamage": 412,
            "magicDamage": 412,
            "physicalDamage": 0,
            "trueDamage": 0,
            "totalDamage": 412,
            "casterParticipantId": 6,
            "wasKillingBlow": false
        },
        ...
    ]
    champion_map: {participantId: championName}
    """
    if not death_recap:
        return "데미지 정보 없음"

    # 캐스터별 총 딜 집계
    caster_damage: dict[int, dict] = {}
    for entry in death_recap:
        cid = entry.get("casterParticipantId")
        if cid not in caster_damage:
            caster_damage[cid] = {
                "total": 0,
                "spells": [],
                "champion": champion_map.get(str(cid), f"참가자{cid}"),
                "killing_blow": False,
            }
        caster_damage[cid]["total"] += entry.get("totalDamage", 0)
        caster_damage[cid]["spells"].append(entry.get("displayName", entry.get("spellName", "?")))
        if entry.get("wasKillingBlow"):
            caster_damage[cid]["killing_blow"] = True

    # 딜량 내림차순 정렬
    sorted_casters = sorted(caster_damage.items(), key=lambda x: x[1]["total"], reverse=True)

    lines = []
    for cid, data in sorted_casters[:3]:  # 상위 3명만
        spell_str = " + ".join(data["spells"][:2])
        kb_marker = " ⚔️킬" if data["killing_blow"] else ""
        lines.append(
            f"  {data['champion']}: {spell_str} → {data['total']:,}딜{kb_marker}"
        )

    return "\n".join(lines)


# 예시 출력:
# Orianna: Command: Shockwave + Command: Attack → 412딜
# Leona: Zenith Blade + Solar Flare → 318딜  ⚔️킬
# Jhin: Curtain Call → 290딜
```

---

## 6. 실제 해설 예시

### 입력 데이터 (T1 vs HLE, KR LCK 2026)

```
한타: 게임 22:14, 바론 근처
결과: T1(블루) 승리, 4킬 vs 1킬
골드 스윙: +3,200골드
오브젝트: Baron Nashor 획득
승률: 54% → 71%
```

### Level 1 출력 예시

```
T1이 집단 싸움에서 4명을 잡아내며 완승을 거뒀습니다! 22분에 벌어진 이 싸움 이후
바론(게임을 유리하게 만들어주는 강력한 버프)까지 획득하며 T1의 승리 가능성이
54%에서 71%로 크게 높아졌습니다.
```

### Level 3 출력 예시

```
22분 14초, 바론 스폰 46초 전에 시작된 이 한타는 T1이 사전 시야 작업을 마친 후
Leona의 Zenith Blade 진입으로 개막됐습니다. Leona가 HLE의 Jinx에 Solar Flare를
정확히 맞추는 순간(22:14.3) Orianna의 즉각적인 Command: Shockwave가 이어지며
Jinx, Zed 2인을 동시에 CC 상태로 묶었고, Jhin이 4번째 총알로 Jinx를 처리했습니다.

특히 주목할 점은 T1의 진입 타이밍입니다. HLE Zed의 R 쿨타임이 32초 남은 상태를
확인하고(22:10 시점 쿨타임 추적 데이터 기준) 의도적으로 이 타이밍을 선택한 것으로
보입니다. Zed의 그림자 귀환이 불가능한 상태에서 진입했기 때문에 생존 탈출 옵션이
없었습니다.

골드 스윙 +3,200골드와 바론 버프 획득이 맞물리며 승률이 17%p 급상승했습니다.
대안 시나리오: HLE가 바론 구덩이 반대편 출구에 와드 1개를 더 배치했다면 T1의
측면 진입 루트가 노출되어 다른 결과가 나올 수 있었습니다.
```

---

## 7. 프롬프트 버전 관리

```
prompts/
├── v1.0/
│   ├── L1_system.txt
│   ├── L2_system.txt
│   ├── L3_system.txt
│   └── L4_system.txt
├── v1.1/  (패치 14.8 이후 메타 반영)
└── CHANGELOG.md

버전 관리 기준:
- 메이저 패치 (새 챔피언·대규모 리워크) → 버전 업
- 해설 품질 피드백 반영 → 서브버전 업
- A/B 테스트: 같은 한타에 v1.0 vs v1.1 동시 생성 → 인간 평가 점수 비교
```

---

## 8. 성능 목표

> **⚫ 미측정 (Gate B 이후)**: Bedrock 실 크레덴셜 연결 후 E2E 레이턴시 측정 예정.
> 현재 mock 모드에서는 응답시간 측정 불가.

| 지표 | 목표 | 측정 방법 | 현재 상태 |
|------|------|---------|---------|
| 레이턴시 L1 | < 1초 | Bedrock invoke 응답시간 | ⚫ 미측정 |
| 레이턴시 L2 | < 2초 | — | ⚫ 미측정 |
| 레이턴시 L3 | < 4초 | — | ⚫ 미측정 |
| 레이턴시 L4 | < 8초 | 비동기 처리 허용 | ⚫ 미측정 |
| 정확도 L3+ | > 90% | deathRecap 챔피언명 정확 매핑 여부 | ⚫ 미측정 |
| 사용자 만족도 | > 4.2/5.0 | 인앱 피드백 (Level별) | ⚫ 미측정 |

---

## 9. 관련 파일

- [[live-stats-api-analysis]] — deathRecap 구조 상세
- [[win-probability-lightgbm]] — Win Probability 스윙 계산
- [[_index]] — 프로젝트 전체 개요 (Neo4j 정적 온톨로지 스키마 포함 — champion·skill·team·player)

> **참고**: Gate A 구현에서는 Amazon Neptune 대신 **Neo4j (로컬 Docker)** 사용.
> `[[neptune-schema]]` 레퍼런스는 목표 아키텍처(AWS 프로덕션) 기준이며, 현재는 Neo4j로 대체.
