# Session 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

에이전트의 상태를 저장하고 복원하는 세션 영속성 모듈. JSON 파일과 Redis 두 가지 백엔드를 지원하며, `StateModule` 패턴과 연동하여 에이전트 전체 상태를 직렬화/역직렬화한다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_session_base.py` | 추상 세션 기반 클래스 |
| `_json_session.py` | JSON 파일 기반 저장 |
| `_redis_session.py` | Redis 기반 저장 |

---

## 핵심 클래스

### `SessionBase`

```python
class SessionBase:
    async def save_session_state(
        session_id: str,
        user_id: str,
        **state_modules_mapping: StateModule
    ) -> None

    async def load_session_state(
        session_id: str,
        user_id: str,
        allow_not_exist: bool = False,
        **state_modules_mapping: StateModule
    ) -> None
```

`**state_modules_mapping`에 `StateModule`을 상속한 객체를 전달하면 자동으로 직렬화/역직렬화.

---

## 지원 백엔드

### JSON 세션

```python
from agentscope.session import JsonSession

session = JsonSession(save_dir="./sessions/")

# 상태 저장
await session.save_session_state(
    session_id="chat_001",
    user_id="user_123",
    agent=my_agent,
    memory=my_memory,
)

# 상태 복원
await session.load_session_state(
    session_id="chat_001",
    user_id="user_123",
    agent=my_agent,
    memory=my_memory,
)
```

파일 구조:
```
sessions/
  └─ user_123/
       └─ chat_001/
            ├─ agent.json
            └─ memory.json
```

### Redis 세션

```python
from agentscope.session import RedisSession

session = RedisSession(
    host="localhost",
    port=6379,
    db=0,
    ttl=3600,   # 세션 만료 시간 (초)
)
```

분산 환경에서 여러 인스턴스가 동일한 세션에 접근할 수 있어 수평 확장에 적합.

---

## StateModule 패턴

`StateModule`을 상속한 모든 컴포넌트는 세션에 저장 가능:

```python
class StateModule:
    def dump_state() -> dict          # 상태를 dict로 직렬화
    def load_state(state: dict)       # dict에서 상태 복원

# StateModule을 상속한 클래스들:
# - AgentBase (에이전트)
# - MemoryBase (메모리)
# - Toolkit (툴킷)
# - KnowledgeBase (지식 베이스)
```

---

## 세션 활용 패턴

### 웹 서비스에서 대화 이어가기

```python
from fastapi import FastAPI
from agentscope.session import JsonSession

app = FastAPI()
session = JsonSession(save_dir="./sessions/")

@app.post("/chat/{session_id}")
async def chat(session_id: str, user_id: str, message: str):
    agent = ReActAgent(name="assistant", ...)
    memory = InMemoryMemory()

    # 이전 세션 복원 (없으면 새로 시작)
    await session.load_session_state(
        session_id=session_id,
        user_id=user_id,
        allow_not_exist=True,
        agent=agent,
        memory=memory,
    )

    response = await agent.reply(Msg(..., content=message))

    # 현재 세션 저장
    await session.save_session_state(
        session_id=session_id,
        user_id=user_id,
        agent=agent,
        memory=memory,
    )

    return {"response": response.content}
```

---

## 모듈 연동

```
Session
  ← Agent      (에이전트 상태 저장/복원)
  ← Memory     (대화 이력 영속성)
  ← Toolkit    (툴킷 상태 저장)
  → StateModule (직렬화 프로토콜)
```
