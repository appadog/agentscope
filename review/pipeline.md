# Pipeline 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

멀티 에이전트 간 메시지 흐름을 오케스트레이션하는 모듈. `MsgHub`로 메시지 브로드캐스트를 관리하고, `ChatRoom`으로 실시간 음성 에이전트 간 이벤트를 라우팅한다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_msghub.py` | 에이전트 간 메시지 허브 |
| `_chat_room.py` | 실시간 음성 채팅룸 |
| `_functional.py` | 함수형 파이프라인 정의 |
| `_class.py` | 클래스 기반 파이프라인 |

---

## 핵심 클래스

### `MsgHub`

```python
class MsgHub:
    participants: list[AgentBase]
    announcement: Msg | list[Msg] | None   # 초기 공지 메시지
    enable_auto_broadcast: bool
    name: str

    # 컨텍스트 매니저
    async def __aenter__() -> MsgHub
    async def __aexit__(...) -> None

    async def broadcast(msg: Msg) -> None  # 수동 브로드캐스트
    def add(new_participant: AgentBase) -> None  # 동적 참가자 추가
```

**동작 방식**: `MsgHub` 컨텍스트 내에서 에이전트가 `reply()`를 호출하면, 그 응답이 자동으로 모든 참가자에게 브로드캐스트된다.

---

### `ChatRoom`

```python
class ChatRoom:
    agents: list[RealtimeAgent]

    async def start(outgoing_queue: asyncio.Queue) -> None
    async def stop() -> None
    def _forward_loop() -> None   # 이벤트 라우팅 루프
```

실시간 에이전트 간 오디오/텍스트 이벤트를 라우팅하는 가상 채팅룸.

---

## 사용 패턴

### 멀티 에이전트 토론

```python
from agentscope.pipeline import MsgHub

moderator = ReActAgent(name="사회자", ...)
agent_a = ReActAgent(name="찬성측", ...)
agent_b = ReActAgent(name="반대측", ...)

async with MsgHub(
    participants=[moderator, agent_a, agent_b],
    announcement=Msg(
        name="system",
        role="system",
        content="AI 윤리에 대해 토론하세요."
    ),
) as hub:
    # 각 에이전트의 응답이 자동으로 모든 참가자에게 전달됨
    topic = Msg(name="user", role="user", content="AI는 인간을 대체할 수 있는가?")
    response = await moderator.reply(topic)
```

### 순서가 있는 대화

```python
async with MsgHub(participants=[agent_a, agent_b]) as hub:
    msg = initial_message
    for _ in range(5):  # 5라운드 토론
        msg = await agent_a.reply(msg)
        msg = await agent_b.reply(msg)
```

### 동적 참가자 추가

```python
async with MsgHub(participants=[agent_a]) as hub:
    # 대화 중에 새 참가자 추가
    hub.add(agent_b)
    await hub.broadcast(Msg(...))
```

---

### 실시간 음성 채팅룸

```python
from agentscope.pipeline import ChatRoom

room = ChatRoom(agents=[voice_agent_1, voice_agent_2])

outgoing_queue = asyncio.Queue()

async with room:
    await room.start(outgoing_queue)
    # 이벤트 루프가 오디오 스트림을 각 에이전트에 라우팅
```

---

## 메시지 흐름

### MsgHub 자동 브로드캐스트

```
agent_a.reply(msg)
  → 응답 생성
  → MsgHub가 응답을 가로챔
  → agent_b.observe(응답)
  → agent_c.observe(응답)
  → 응답 반환
```

### ChatRoom 이벤트 라우팅

```
입력 오디오 스트림
  ↓
ChatRoom._forward_loop()
  ├─ RealtimeAgent_1에 오디오 전달
  └─ RealtimeAgent_2에 오디오 전달
      ↓
  각 에이전트의 응답 → outgoing_queue
```

---

## 함수형 파이프라인

```python
from agentscope.pipeline import sequential, parallel, ifelse

# 순차 실행
result = await sequential(
    [agent_a, agent_b, agent_c],
    initial_msg
)

# 병렬 실행
results = await parallel(
    [agent_a, agent_b],
    initial_msg
)

# 조건 분기
result = await ifelse(
    condition=lambda msg: "urgent" in msg.content,
    true_agent=urgent_agent,
    false_agent=normal_agent,
    msg=initial_msg
)
```

---

## 모듈 연동

```
Pipeline
  ← Agent      (MsgHub 참가자로 등록)
  ← Realtime   (ChatRoom에서 실시간 에이전트 운영)
  → Message    (Msg 브로드캐스트)
```
