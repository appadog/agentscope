# 배포 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/deployment/planning_agent/`

---

## planning_agent — 프로덕션 서버 배포

**파일**: `main.py`, `tool.py`, `test_post.py`, `README.md`

**목적**: 플래닝 에이전트를 Quart(비동기 Flask) 기반 HTTP 서버로 배포하는 프로덕션 패턴.

---

## 서버 아키텍처

```
클라이언트 (HTTP POST)
  ↓
Quart 서버
  ↓ /chat_endpoint
JSONSession (세션 복원)
  ↓
MetaPlannerAgent.reply()
  ↓
stream_printing_messages (SSE 스트리밍)
  ↓
클라이언트 (Server-Sent Events)
```

---

## 핵심 코드

### 서버 엔드포인트

```python
from quart import Quart, request, Response
from agentscope.session import JSONSession
from agentscope.pipeline import stream_printing_messages

app = Quart(__name__)
session = JSONSession(save_dir="./sessions/")

@app.route("/chat_endpoint", methods=["POST"])
async def chat_endpoint():
    data = await request.get_json()
    user_id = data["user_id"]
    session_id = data["session_id"]
    user_input = data["user_input"]

    # 에이전트 및 메모리 초기화
    agent = create_planner_agent()
    memory = InMemoryMemory()

    # 이전 세션 복원
    await session.load_session_state(
        session_id=session_id,
        user_id=user_id,
        allow_not_exist=True,
        agent=agent,
        memory=memory,
    )

    user_msg = Msg(name="user", content=user_input, role="user")

    async def generate():
        async for msg, is_last in stream_printing_messages(
            agents=[agent],
            coroutine_task=agent(user_msg),
        ):
            # SSE 포맷으로 스트리밍
            yield f"data: {msg.model_dump_json()}\n\n"

        # 세션 저장
        await session.save_session_state(
            session_id=session_id,
            user_id=user_id,
            agent=agent,
            memory=memory,
        )

    return Response(
        generate(),
        mimetype="text/event-stream",
        headers={"Cache-Control": "no-cache"},
    )
```

### 클라이언트 테스트

```python
# test_post.py
import httpx

async def test():
    async with httpx.AsyncClient() as client:
        async with client.stream(
            "POST",
            "http://localhost:5000/chat_endpoint",
            json={
                "user_id": "alice",
                "session_id": "session_001",
                "user_input": "파이썬으로 웹 스크래퍼를 만들어줘",
            },
        ) as response:
            async for line in response.aiter_lines():
                if line.startswith("data: "):
                    print(json.loads(line[6:])["content"])
```

---

## 구성 요소

| 컴포넌트 | 역할 |
|----------|------|
| Quart | 비동기 HTTP 서버 프레임워크 |
| JSONSession | 세션 상태 영속성 |
| MetaPlannerAgent | 계층적 플래닝 에이전트 |
| stream_printing_messages | SSE 스트리밍 |
| create_worker 툴 | 동적 워커 에이전트 생성 |

---

## 배포 시 고려사항

1. **세션 격리**: `user_id + session_id` 조합으로 멀티 유저 지원
2. **스트리밍**: SSE로 LLM 응답을 실시간 전송, 사용자 체감 대기시간 감소
3. **상태 영속성**: 서버 재시작 후에도 이전 대화 복원 가능
4. **비동기 처리**: Quart의 asyncio 기반으로 다수 동시 요청 처리

---

## 확장 방안

```python
# Redis 세션으로 교체 (수평 확장)
from agentscope.session import RedisSession

session = RedisSession(
    host="redis-cluster",
    port=6379,
    ttl=3600,
)

# 여러 Quart 인스턴스가 동일한 Redis 세션 공유
```
