# 워크플로우 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/workflows/`

멀티 에이전트 협력 패턴을 시연하는 4가지 워크플로우 예제.

---

## 1. multiagent_conversation — 그룹 대화

**파일**: `main.py`

여러 에이전트가 공통 주제로 그룹 대화를 나누는 패턴.

```python
from agentscope.pipeline import MsgHub, sequential_pipeline

# 에이전트 생성
host = ReActAgent(name="사회자", sys_prompt="그룹 대화를 진행하세요.", ...)
agent_a = ReActAgent(name="Alice", sys_prompt="친절한 성격을 가지고 있습니다.", ...)
agent_b = ReActAgent(name="Bob", sys_prompt="분석적인 성격을 가지고 있습니다.", ...)
agent_c = ReActAgent(name="Carol", sys_prompt="창의적인 성격을 가지고 있습니다.", ...)

# MsgHub: 모든 에이전트의 응답이 자동으로 전체 참가자에게 전달
async with MsgHub(
    participants=[host, agent_a, agent_b, agent_c],
    announcement=Msg(
        name="system",
        role="system",
        content="각자 자기소개를 해주세요.",
    ),
) as hub:
    # 순차 진행: 사회자 → Alice → Bob → Carol
    await sequential_pipeline([host, agent_a, agent_b, agent_c], initial_msg)
```

**핵심 동작**:
- `MsgHub` 내에서 에이전트가 응답하면 다른 모든 참가자에게 자동 전달
- `announcement` 메시지는 컨텍스트 시작 시 전달되는 공지

---

## 2. multiagent_debate — 구조화된 토론

**파일**: `main.py`

찬성/반대 두 에이전트가 토론하고, 심판 에이전트가 결론을 내리는 패턴.

```python
class JudgeOutput(BaseModel):
    winner: Literal["찬성", "반대", "무승부"]
    reason: str
    is_finished: bool

affirmative = ReActAgent(
    name="찬성측",
    sys_prompt="주어진 주제에 찬성 입장을 논리적으로 주장하세요.",
    ...
)
negative = ReActAgent(
    name="반대측",
    sys_prompt="주어진 주제에 반대 입장을 논리적으로 주장하세요.",
    ...
)
judge = ReActAgent(
    name="심판",
    sys_prompt="토론 내용을 평가하고 승자를 결정하세요.",
    ...
)

topic = Msg(name="user", content="AI는 인간의 일자리를 대체할 것인가?", role="user")

async with MsgHub(participants=[affirmative, negative, judge]) as hub:
    for round_num in range(3):
        affirmative_msg = await affirmative.reply(topic)
        negative_msg = await negative.reply(affirmative_msg)

        # 심판이 구조화된 출력으로 결론 판정
        judgment = await judge.reply(
            negative_msg,
            structured_model=JudgeOutput,
        )

        if judgment.metadata.is_finished:
            break
```

---

## 3. multiagent_concurrent — 병렬 실행

**파일**: `main.py`

여러 에이전트를 동시에 실행하여 처리 시간을 단축하는 패턴.

```python
import asyncio
from agentscope.pipeline import fanout_pipeline

# 병렬 실행 (asyncio.gather 내부 사용)
start = time.time()
results = await fanout_pipeline(
    agents=[agent1, agent2, agent3, agent4],
    msg=question,
    sequential=False,   # 병렬 모드
)
parallel_time = time.time() - start

# 순차 실행 (비교용)
start = time.time()
results = await fanout_pipeline(
    agents=[agent1, agent2, agent3, agent4],
    msg=question,
    sequential=True,    # 순차 모드
)
sequential_time = time.time() - start

print(f"병렬: {parallel_time:.2f}s vs 순차: {sequential_time:.2f}s")
```

**측정 결과 예시**: 4개 에이전트 병렬 실행 시 순차 대비 ~3.5x 빠름.

---

## 4. multiagent_realtime — 실시간 음성 멀티 에이전트

**파일**: `run_server.py`, `multi_agent.html`

여러 음성 에이전트가 `ChatRoom`에서 함께 대화하는 실시간 음성 패턴.

```python
from agentscope.pipeline import ChatRoom

# 여러 실시간 에이전트 생성
agent1 = RealtimeAgent(
    name="Alice",
    realtime_model=OpenAIRealtimeModel(...),
    instructions="당신은 Alice입니다.",
)
agent2 = RealtimeAgent(
    name="Bob",
    realtime_model=GeminiRealtimeModel(...),
    instructions="당신은 Bob입니다.",
)

# ChatRoom에서 에이전트 이벤트 라우팅
room = ChatRoom(agents=[agent1, agent2])

@app.websocket("/ws/{user_id}")
async def ws_endpoint(websocket: WebSocket, user_id: str):
    outgoing_queue = asyncio.Queue()

    async with room:
        await room.start(outgoing_queue)

        # 클라이언트 오디오 → 에이전트들에게 전달
        async for event in websocket.iter_json():
            await room.broadcast(event)
```

---

## 워크플로우 패턴 비교

| 패턴 | 에이전트 관계 | 통신 방식 | 적합한 시나리오 |
|------|------------|----------|----------------|
| 그룹 대화 | 평등 (P2P) | MsgHub 브로드캐스트 | 아이디어 토론, 집단 창작 |
| 토론 | 역할 분리 | 순차 + 심판 | 의사결정, 검증 |
| 병렬 실행 | 독립 | fanout_pipeline | 대량 처리, 앙상블 |
| 실시간 음성 | 공존 | ChatRoom 이벤트 | 음성 비서, 회의 |
