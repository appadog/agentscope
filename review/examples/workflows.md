# 워크플로우 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/workflows/`

---

## 1. multiagent_conversation — 그룹 대화

**파일**: `main.py`

`MsgHub`로 여러 에이전트가 공통 대화 컨텍스트를 공유하는 패턴.

**실제 코드:**

```python
# examples/workflows/multiagent_conversation/main.py
def create_participant_agent(name, age, career, character) -> ReActAgent:
    return ReActAgent(
        name=name,
        sys_prompt=(
            f"You're a {age}-year-old {career} named {name} and you're "
            f"a {character} person."
        ),
        model=DashScopeChatModel(
            model_name="qwen-max",
            api_key=os.environ["DASHSCOPE_API_KEY"],
            stream=True,
        ),
        formatter=DashScopeMultiAgentFormatter(),  # 멀티에이전트용 포매터
    )

alice = create_participant_agent("Alice", 30, "teacher", "friendly")
bob   = create_participant_agent("Bob", 14, "student", "rebellious")
charlie = create_participant_agent("Charlie", 28, "doctor", "thoughtful")

async with MsgHub(
    participants=[alice, bob, charlie],
    announcement=Msg("system", "Now you meet each other with a brief self-introduction.", "system"),
) as hub:
    # 순차 파이프라인으로 한 번씩 발언
    await sequential_pipeline([alice, bob, charlie])

    # Bob 퇴장 처리
    hub.delete(bob)
    await hub.broadcast(
        Msg("bob", "I have to start my homework now, see you later!", "assistant")
    )
    await alice()
    await charlie()
```

**핵심 포인트:**
- `DashScopeMultiAgentFormatter` — 여러 에이전트 이름이 프롬프트에 등장할 때 필수
- `hub.delete(bob)` — 실행 중 참가자 제거
- `hub.broadcast()` — 모든 참가자에게 메시지 전달

---

## 2. multiagent_debate — 멀티에이전트 토론

**파일**: `main.py`

두 토론자(Alice, Bob)와 사회자(Aggregator)가 수학 문제를 토론으로 해결.

**실제 코드:**

```python
# examples/workflows/multiagent_debate/main.py
topic = "The two circles are externally tangent ... How many times will circle A revolve?"

def create_solver_agent(name: str) -> ReActAgent:
    return ReActAgent(
        name=name,
        sys_prompt=f"You're a debater named {name}. ... The debate topic is: {topic}",
        model=DashScopeChatModel(model_name="qwen-max", ...),
        formatter=DashScopeChatFormatter(),
    )

alice, bob = [create_solver_agent(name) for name in ["Alice", "Bob"]]

moderator = ReActAgent(
    name="Aggregator",
    sys_prompt="You're a moderator. ...",
    model=DashScopeChatModel(model_name="qwen-max", ...),
    formatter=DashScopeMultiAgentFormatter(),  # 멀티에이전트 포매터
)

class JudgeModel(BaseModel):
    finished: bool = Field(description="Whether the debate is finished.")
    correct_answer: str | None = Field(description="The correct answer, if finished.", default=None)

async def run_multiagent_debate():
    while True:
        # Alice, Bob은 MsgHub 안에서 서로의 메시지를 봄
        async with MsgHub(participants=[alice, bob, moderator]):
            await alice(Msg("user", "You are affirmative side...", "user"))
            await bob(Msg("user", "You are negative side...", "user"))

        # 사회자는 MsgHub 밖에서 판단 (Alice, Bob은 판결 메시지를 못 봄)
        msg_judge = await moderator(
            Msg("user", "Have the debate finished, and can you get the correct answer?", "user"),
            structured_model=JudgeModel,
        )

        if msg_judge.metadata.get("finished"):
            print("Correct answer:", msg_judge.metadata.get("correct_answer"))
            break
```

**핵심 패턴:**
- `MsgHub` 안: 토론자들이 서로의 메시지를 공유
- `MsgHub` 밖: 사회자만 판결 — 토론자들은 판결 내용 모름
- `JudgeModel`로 종료 조건을 구조화 출력으로 판별

---

## 3. multiagent_concurrent — 병렬 동시 실행

**파일**: `main.py`

`asyncio.gather` 와 `fanout_pipeline`으로 여러 에이전트를 동시 실행.

**실제 코드:**

```python
# examples/workflows/multiagent_concurrent/main.py
class ExampleAgent(AgentBase):
    def __init__(self, name: str):
        super().__init__()
        self.name = name

    async def reply(self, *args, **kwargs) -> Msg:
        start_time = datetime.now()
        await self.print(Msg(self.name, f"begins at {start_time.strftime('%H:%M:%S.%f')}", "assistant"))

        await asyncio.sleep(np.random.choice([2, 3, 4]))  # 랜덤 지연

        end_time = datetime.now()
        msg = Msg(
            self.name,
            f"finishes at {end_time.strftime('%H:%M:%S.%f')}",
            "user",
            metadata={"time": (end_time - start_time).total_seconds()},
        )
        return msg

alice = ExampleAgent("Alice")
bob   = ExampleAgent("Bob")
chalice = ExampleAgent("Chalice")

# 방법 1: asyncio.gather 직접 사용
await asyncio.gather(*[alice(), bob(), chalice()])

# 방법 2: fanout_pipeline
collected_res = await fanout_pipeline(
    agents=[alice, bob, chalice],
    enable_gather=True,
)

# 결과 분석
avg_time = np.mean([res.metadata["time"] for res in collected_res])
print(f"Average time: {avg_time} seconds")
```

**핵심 포인트:**
- `asyncio.gather()` — 파이썬 표준 비동기 병렬 실행
- `fanout_pipeline(enable_gather=True)` — AgentScope 래퍼 (결과 수집 포함)
- `metadata` 필드로 부가 정보 전달

---

## 4. multiagent_realtime — 실시간 멀티에이전트

**파일**: `run_server.py`

WebSocket 기반 실시간 음성 멀티에이전트 시스템.

```python
# run_server.py (개요)
from agentscope.agent import RealtimeAgent

agent = RealtimeAgent(
    name="voice_assistant",
    model=DashScopeRealtimeModel(...),
)
# WebSocket 서버로 실시간 오디오 스트리밍
```

---

## 패턴 비교

| 워크플로우 | 패턴 | 에이전트 관계 | 포매터 |
|-----------|------|-------------|--------|
| multiagent_conversation | MsgHub + sequential_pipeline | 순차 발언, 공유 컨텍스트 | `DashScopeMultiAgentFormatter` |
| multiagent_debate | MsgHub 안/밖 분리 | 토론자 vs 사회자 | MultiAgent + Single 혼합 |
| multiagent_concurrent | asyncio.gather / fanout_pipeline | 독립 병렬 실행 | 각자 독립 |
| multiagent_realtime | WebSocket | 실시간 음성 | RealtimeModel |
