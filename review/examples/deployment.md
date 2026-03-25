# 배포 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/deployment/planning_agent/`

---

## planning_agent — 프로덕션 SSE 스트리밍 서버

**파일**: `main.py` (Quart 서버), `tool.py` (worker 생성 툴), `test_post.py` (테스트 클라이언트)

**실제 코드 (`main.py`):**

```python
# examples/deployment/planning_agent/main.py
from quart import Quart, Response, request
from agentscope.pipeline import stream_printing_messages
from agentscope.session import JSONSession

app = Quart(__name__)


async def handle_input(msg: Msg, user_id: str, session_id: str) -> AsyncGenerator[str, None]:
    toolkit = Toolkit()
    toolkit.register_tool_function(create_worker)  # tool.py의 worker 생성 함수

    session = JSONSession(save_dir="./sessions")

    agent = ReActAgent(
        name="Friday",
        sys_prompt="""You are Friday, a multifunctional agent...
## Core Mission
Break down complicated tasks into subtasks, create worker agents to finish subtasks.
### Important Constraints
1. DO NOT TRY TO SOLVE THE SUBTASKS DIRECTLY yourself.
2. Always follow the plan sequence.
""",
        model=DashScopeChatModel(
            model_name="qwen3-max",
            api_key=os.environ.get("DASHSCOPE_API_KEY"),
        ),
        formatter=DashScopeChatFormatter(),
        toolkit=toolkit,
    )

    # 이전 세션 복원
    await session.load_session_state(
        session_id=f"{user_id}-{session_id}",
        agent=agent,
    )

    # SSE 스트리밍
    async for msg, _ in stream_printing_messages(
        agents=[agent],
        coroutine_task=agent(msg),
    ):
        data = json.dumps(msg.to_dict(), ensure_ascii=False)
        yield f"data: {data}\n\n"

    # 세션 저장
    await session.save_session_state(
        session_id=f"{user_id}-{session_id}",
        agent=agent,
    )


@app.route("/chat_endpoint", methods=["POST"])
async def chat_endpoint() -> Response:
    data = await request.get_json()
    user_id = data.get("user_id")
    session_id = data.get("session_id")
    user_input = data.get("user_input")

    if not user_id or not session_id:
        return Response(f"user_id and session_id are required", status=400)

    return Response(
        handle_input(Msg("user", user_input, "user"), user_id, session_id),
        mimetype="text/event-stream",
    )


if __name__ == "__main__":
    app.run(port=5000, debug=True)
```

**worker 툴 (`tool.py`):**

```python
# examples/deployment/planning_agent/tool.py
async def create_worker(description: str, system_prompt: str) -> str:
    """Creates a worker agent to handle a specific sub-task."""
    worker = ReActAgent(
        name="worker",
        sys_prompt=system_prompt,
        model=DashScopeChatModel(...),
        formatter=DashScopeChatFormatter(),
        toolkit=Toolkit(),  # 내장 툴 등록
    )
    result = await worker(Msg("user", description, "user"))
    return result.get_text_content()
```

**테스트 클라이언트 (`test_post.py`):**

```python
# examples/deployment/planning_agent/test_post.py
import httpx

async def main():
    async with httpx.AsyncClient() as client:
        async with client.stream(
            "POST",
            "http://localhost:5000/chat_endpoint",
            json={
                "user_id": "alice",
                "session_id": "session_001",
                "user_input": "Write a Python script that analyzes...",
            },
        ) as response:
            async for line in response.aiter_lines():
                if line.startswith("data: "):
                    data = json.loads(line[6:])
                    print(data.get("content", ""))
```

---

## 아키텍처

```
클라이언트 (HTTP POST)
    ↓
Quart 서버 (/chat_endpoint)
    ↓
JSONSession.load_session_state()
    ↓
ReActAgent (메타 플래너 Friday)
    ↓ tool 호출
create_worker(description, system_prompt)
    ↓
Worker ReActAgent (하위 태스크 처리)
    ↓
stream_printing_messages → SSE 스트리밍
    ↓
JSONSession.save_session_state()
```

**핵심 패턴:**
- `JSONSession` — 사용자별 에이전트 상태 파일 저장 (`./sessions/{user_id}-{session_id}.json`)
- `stream_printing_messages` — 에이전트 내부 메시지를 실시간으로 클라이언트에 전달
- `create_worker` 툴 — 메타 플래너가 동적으로 하위 에이전트 생성
- SSE 포맷: `data: {json}\n\n`
