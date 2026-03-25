# 게임 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/game/werewolves/`

---

## werewolves — 늑대인간 게임

**파일**: `main.py`, `game.py`, `prompt.py`, `structured_model.py`, `utils.py`, `README.md`

**목적**: 멀티 에이전트 시스템의 복잡한 역할 기반 상호작용과 전략적 의사소통을 시연하는 시뮬레이션.

---

## 게임 구성

**플레이어 (10명)**:

| 역할 | 수 | 능력 |
|------|---|------|
| 늑대인간 | 3 | 밤에 마을 주민 제거 |
| 마을 주민 | 3 | 토론 및 투표 |
| 예언자 | 1 | 매 밤 한 명의 역할 확인 |
| 마녀 | 1 | 독약/해독약 각 1회 사용 |
| 사냥꾼 | 1 | 탈락 시 한 명 지목 제거 |
| 사회자 | 1 (AI) | 게임 진행 관리 |

---

## 아키텍처

```python
# 역할별 시스템 프롬프트
PROMPTS = {
    "werewolf": "당신은 늑대인간입니다. 다른 늑대인간과 협력하여 마을 주민을 제거하세요...",
    "villager": "당신은 마을 주민입니다. 토론을 통해 늑대인간을 찾아내세요...",
    "seer": "당신은 예언자입니다. 매 밤 한 명의 진짜 역할을 확인할 수 있습니다...",
    "witch": "당신은 마녀입니다. 독약과 해독약을 전략적으로 사용하세요...",
    "hunter": "당신은 사냥꾼입니다. 제거될 때 한 명을 함께 제거할 수 있습니다...",
}

# 플레이어 에이전트 생성
players = {
    name: ReActAgent(
        name=name,
        sys_prompt=PROMPTS[role],
        model=DashScopeChatModel(...),
    )
    for name, role in assignments.items()
}
```

---

## 게임 흐름 (상태 머신)

```
초기화 (역할 배분)
  ↓
밤 페이즈 (NIGHT)
  ├─ 늑대인간 팀 비밀 투표 → 희생자 결정
  ├─ 예언자 → 1명 역할 확인
  └─ 마녀 → 독약/해독약 사용 결정
  ↓
낮 페이즈 (DAY)
  ├─ 밤 결과 발표
  ├─ 전체 토론 (MsgHub 브로드캐스트)
  └─ 투표 → 추방 대상 결정
  ↓
승리 조건 확인
  ├─ 늑대인간 수 ≥ 마을 주민 수 → 늑대인간 승리
  └─ 늑대인간 전멸 → 마을 주민 승리
  ↓
(반복 또는 게임 종료)
```

---

## 구조화 출력 활용

```python
class VoteOutput(BaseModel):
    target: str           # 투표 대상 플레이어 이름
    reason: str           # 투표 이유

class WitchDecision(BaseModel):
    use_poison: bool
    poison_target: str | None
    use_antidote: bool

# 각 플레이어의 행동을 검증된 구조로 수집
vote = await player.reply(
    discussion_msg,
    structured_model=VoteOutput,
)
```

---

## 세션 영속성

```python
from agentscope.session import JSONSession

session = JSONSession(save_dir="./game_saves/")

# 게임 중단 후 이어서 플레이 가능
await session.save_session_state(
    session_id="game_001",
    user_id="host",
    **{name: agent for name, agent in players.items()},
    game_state=game_state_module,
)
```

---

## AgentScope 활용 포인트

| 기능 | 활용 방식 |
|------|----------|
| `MsgHub` | 낮 페이즈 전체 토론 브로드캐스트 |
| `ReActAgent` | 각 플레이어의 전략적 추론 |
| 구조화 출력 | 투표/행동 결과의 정확한 파싱 |
| `JSONSession` | 게임 상태 저장/복원 |
| 역할별 프롬프트 | 각 역할의 고유한 행동 패턴 구현 |
