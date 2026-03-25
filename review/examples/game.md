# 게임 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/game/werewolves/`

---

## werewolves — 늑대인간 게임

9명이 참여하는 완전한 늑대인간(마피아) 게임 시뮬레이션. 구조화 출력으로 각 역할의 행동을 제어.

**파일 구조:**
```
werewolves/
├── main.py           — 게임 진입점 (에이전트 생성 + 역할 배정)
├── game.py           — 게임 로직 (밤/낮 단계 구현)
├── structured_model.py — 역할별 Pydantic 출력 모델
├── prompt.py         — 영어/한국어 프롬프트 (EnglishPrompts, ChinesePrompts)
└── utils.py          — EchoAgent, majority_vote, Players 클래스
```

---

## structured_model.py — 역할별 구조화 모델

**실제 코드:**

```python
# examples/game/werewolves/structured_model.py
class DiscussionModel(BaseModel):
    """낮 토론 단계 출력 — 합의 여부"""
    reach_agreement: bool = Field(
        description="Whether you have reached an agreement or not"
    )


def get_vote_model(agents: list[AgentBase]) -> type[BaseModel]:
    """에이전트 목록에서 동적으로 투표 모델 생성"""
    class VoteModel(BaseModel):
        vote: Literal[tuple(_.name for _ in agents)] = Field(  # type: ignore
            description="The name of the player you want to vote for"
        )
    return VoteModel


class WitchResurrectModel(BaseModel):
    """마녀 부활 포션 사용 여부"""
    resurrect: bool = Field(description="Whether you want to resurrect the player")


def get_poison_model(agents: list[AgentBase]) -> type[BaseModel]:
    """마녀 독 포션 모델 — 사용 여부 + 대상"""
    class WitchPoisonModel(BaseModel):
        poison: bool = Field(description="Do you want to use the poison potion")
        name: Literal[tuple(_.name for _ in agents)] | None = Field(
            description="The name of the player to poison, or None",
            default=None,
        )

        @model_validator(mode="before")
        def clear_name_if_no_poison(cls, values: dict) -> dict:
            """독을 사용하지 않으면 name을 None으로 강제"""
            if isinstance(values, dict) and not values.get("poison"):
                values["name"] = None
            return values

    return WitchPoisonModel


def get_seer_model(agents: list[AgentBase]) -> type[BaseModel]:
    """예언자 조사 대상 선택"""
    class SeerModel(BaseModel):
        name: Literal[tuple(_.name for _ in agents)] = Field(
            description="The name of the player you want to check"
        )
    return SeerModel


def get_hunter_model(agents: list[AgentBase]) -> type[BaseModel]:
    """사냥꾼 총격 여부 + 대상"""
    class HunterModel(BaseModel):
        shoot: bool = Field(description="Whether you want to use the shooting ability")
        name: Literal[tuple(_.name for _ in agents)] | None = Field(
            description="The name of the player to shoot, or None",
            default=None,
        )

        @model_validator(mode="before")
        def clear_name_if_no_shoot(cls, values: dict) -> dict:
            if isinstance(values, dict) and not values.get("shoot"):
                values["name"] = None
            return values

    return HunterModel
```

**핵심 설계 패턴:**
- `get_vote_model(players.current_alive)` — 살아있는 플레이어로 `Literal` 타입을 동적 생성 → 죽은 플레이어에게 투표 불가
- `@model_validator` — 조건부 필드 검증 (독/총 미사용 시 대상 자동 None)

---

## game.py — 게임 흐름

**실제 코드 (핵심 부분):**

```python
# examples/game/werewolves/game.py
async def hunter_stage(hunter_agent: ReActAgent, players: Players) -> str | None:
    """사냥꾼 단계 (밤에 죽거나 낮에 추방될 때 발동)"""
    msg_hunter = await hunter_agent(
        await moderator(Prompts.to_hunter.format(name=hunter_agent.name)),
        structured_model=get_hunter_model(players.current_alive),
    )
    if msg_hunter.metadata.get("shoot"):
        return msg_hunter.metadata.get("name", None)
    return None


async def werewolves_game(agents: list[ReActAgent]) -> None:
    """메인 게임 루프"""
    assert len(agents) == 9, "The werewolf game needs exactly 9 players."

    players = Players()
    healing, poison = True, True  # 마녀 포션 상태
    # ...
```

**`EchoAgent` (utils.py):** 게임 진행자(moderator) 역할 — TTS 옵션도 포함

```python
class EchoAgent(AgentBase):
    """메시지를 그대로 에코하는 진행자 에이전트 (TTS 옵션 포함)"""
    # tts_model=DashScopeTTSModel(...) 옵션으로 음성 진행 가능
```

---

## 게임 역할 구성 (9명)

| 역할 | 수 | 구조화 모델 | 행동 |
|------|-----|-----------|------|
| 늑대인간 (Werewolf) | 3 | `get_vote_model()` | 밤에 희생자 선택 |
| 마을 주민 (Villager) | 2 | `VoteModel` | 낮에 투표 |
| 예언자 (Seer) | 1 | `get_seer_model()` | 밤에 한 명 정체 확인 |
| 마녀 (Witch) | 1 | `WitchResurrectModel`, `get_poison_model()` | 부활/독 포션 |
| 사냥꾼 (Hunter) | 1 | `get_hunter_model()` | 죽을 때 총격 |
| 수호자 (Guard) | 1 | `get_vote_model()` | 밤에 한 명 보호 |

---

## MsgHub 활용 패턴

```python
# 늑대인간 밤 회의 — 늑대인간끼리만 소통
async with MsgHub(participants=werewolves):
    await sequential_pipeline(werewolves)

# 낮 토론 — 모든 생존자
async with MsgHub(participants=players.current_alive):
    for _ in range(MAX_DISCUSSION_ROUND):
        result = await fanout_pipeline(players.current_alive, enable_gather=False)
        # DiscussionModel로 합의 여부 체크
```
