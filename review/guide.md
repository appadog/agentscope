# AgentScope 사용 가이드

> [← 요약으로 돌아가기](./summary.md)

단계별로 AgentScope를 익히는 순서 가이드. 각 단계는 이전 단계를 기반으로 하므로 순서대로 진행하는 것을 권장한다.

---

## STEP 1 — 설치 및 기본 에이전트 실행

### 설치

```bash
# 기본 설치
pip install agentscope

# 전체 기능 설치 (RAG, 메모리, 평가 등)
pip install "agentscope[full]"

# 필요한 기능만 선택 설치
pip install "agentscope[memory,rag]"
```

### 첫 번째 에이전트

```python
import asyncio
from agentscope.agent import ReActAgent
from agentscope.model import OpenAIChatModel
from agentscope.formatter import OpenAIChatFormatter
from agentscope.message import Msg

async def main():
    agent = ReActAgent(
        name="assistant",
        sys_prompt="당신은 도움이 되는 AI 어시스턴트입니다.",
        model=OpenAIChatModel(
            model_name="gpt-4o-mini",
            api_key="YOUR_API_KEY",
        ),
        formatter=OpenAIChatFormatter(),
    )

    msg = Msg(name="user", content="안녕하세요!", role="user")
    response = await agent.reply(msg)
    print(response.content)

asyncio.run(main())
```

> 참고 예제: [`examples/agent/react_agent/`](../examples/agent/react_agent/)
> 모듈 문서: [agent](./agent.md) | [model](./model.md) | [message](./message.md)

---

## STEP 2 — 모델 제공자 선택

사용할 LLM에 맞는 모델 클래스와 포매터를 조합한다.

| 제공자 | 모델 클래스 | 포매터 클래스 |
|--------|-----------|------------|
| OpenAI | `OpenAIChatModel` | `OpenAIChatFormatter` |
| Anthropic | `AnthropicChatModel` | `AnthropicChatFormatter` |
| Google Gemini | `GeminiChatModel` | `GeminiChatFormatter` |
| 알리바바 DashScope (Qwen) | `DashScopeChatModel` | `DashScopeChatFormatter` |
| 로컬 Ollama | `OllamaChatModel` | `OllamaChatFormatter` |

```python
# DashScope (Qwen) 사용 예시
from agentscope.model import DashScopeChatModel
from agentscope.formatter import DashScopeChatFormatter

agent = ReActAgent(
    name="assistant",
    model=DashScopeChatModel(
        model_name="qwen-max",
        api_key="YOUR_DASHSCOPE_KEY",
    ),
    formatter=DashScopeChatFormatter(),
)

# Anthropic Claude 사용 예시
from agentscope.model import AnthropicChatModel
from agentscope.formatter import AnthropicChatFormatter

agent = ReActAgent(
    name="assistant",
    model=AnthropicChatModel(
        model_name="claude-sonnet-4-6",
        api_key="YOUR_ANTHROPIC_KEY",
    ),
    formatter=AnthropicChatFormatter(),
)
```

> 모듈 문서: [model](./model.md) | [formatter](./formatter.md)

---

## STEP 3 — 툴 추가

에이전트에게 외부 기능(코드 실행, 파일 조작, 웹 검색 등)을 부여한다.

```python
from agentscope.tool import Toolkit
from agentscope.tool import execute_python_code, execute_shell_command

# 툴킷 생성
toolkit = Toolkit()

# 내장 툴 등록
toolkit.register_tool_function(execute_python_code)
toolkit.register_tool_function(execute_shell_command)

# 커스텀 툴 등록
def get_weather(city: str) -> str:
    """지정한 도시의 날씨를 반환합니다.

    Args:
        city: 도시 이름
    """
    return f"{city}의 날씨는 맑음입니다."  # 실제 API 연동 가능

toolkit.register_tool_function(get_weather)

# 에이전트에 툴킷 연결
agent = ReActAgent(
    name="assistant",
    model=...,
    formatter=...,
    toolkit=toolkit,
)
```

> 참고 예제: [`examples/agent/react_agent/`](../examples/agent/react_agent/)
> 모듈 문서: [tool](./tool.md)

---

## STEP 4 — 메모리 추가

대화 이력을 유지하여 맥락 있는 대화를 가능하게 한다.

```python
from agentscope.memory import InMemoryMemory

# 기본 인메모리 (프로세스 종료 시 소멸)
agent = ReActAgent(
    name="assistant",
    memory=InMemoryMemory(),
    ...
)

# 대화 루프
history = []
while True:
    user_input = input("You: ")
    if user_input == "exit":
        break

    msg = Msg(name="user", content=user_input, role="user")
    response = await agent.reply(msg)
    print(f"Assistant: {response.content}")
```

### 메모리 압축 (긴 대화용)

```python
from agentscope.memory import InMemoryMemory

agent = ReActAgent(
    name="assistant",
    memory=InMemoryMemory(),
    compression_config=ReActAgent.CompressionConfig(
        enable=True,
        trigger_threshold=3000,  # 3000 토큰 초과 시 자동 압축
        keep_recent=5,           # 최근 5개 메시지는 보존
    ),
    ...
)
```

> 참고 예제: [`examples/functionality/short_term_memory/`](../examples/functionality/short_term_memory/)
> 모듈 문서: [memory](./memory.md)

---

## STEP 5 — 세션 영속성 (대화 저장/복원)

서버 재시작 후에도 이전 대화를 이어갈 수 있다.

```python
from agentscope.session import JsonSession
from agentscope.memory import InMemoryMemory

session = JsonSession(save_dir="./sessions/")

async def chat(user_id: str, session_id: str, message: str) -> str:
    agent = ReActAgent(name="assistant", memory=InMemoryMemory(), ...)

    # 이전 대화 복원
    await session.load_session_state(
        session_id=session_id,
        user_id=user_id,
        allow_not_exist=True,  # 처음이면 새로 시작
        agent=agent,
    )

    response = await agent.reply(Msg(name="user", content=message, role="user"))

    # 현재 대화 저장
    await session.save_session_state(
        session_id=session_id,
        user_id=user_id,
        agent=agent,
    )

    return response.content
```

> 참고 예제: [`examples/functionality/session_with_sqlite/`](../examples/functionality/session_with_sqlite/)
> 모듈 문서: [session](./session.md)

---

## STEP 6 — RAG (문서 기반 답변)

문서를 읽어서 에이전트가 지식으로 활용할 수 있게 한다.

```python
from agentscope.rag import SimpleKnowledge, TextReader
from agentscope.rag._store._qdrant_store import QdrantStore
from agentscope.embedding import OpenAIEmbeddingModel

# 임베딩 모델 및 벡터 DB 설정
embedding_model = OpenAIEmbeddingModel(
    model_name="text-embedding-3-small",
    api_key="YOUR_API_KEY",
)
store = QdrantStore(
    collection_name="my_docs",
    embedding_dim=1536,
    path="./qdrant_db/",  # 로컬 저장
)

# 지식 베이스 생성
knowledge = SimpleKnowledge(
    embedding_store=store,
    embedding_model=embedding_model,
)

# 문서 추가 (최초 1회)
reader = TextReader(chunk_size=512, chunk_overlap=50)
docs = await reader("./my_documents/")
await knowledge.add_documents(docs)

# 에이전트에 연결 (자동으로 retrieve_knowledge 툴 사용)
agent = ReActAgent(
    name="assistant",
    knowledge_base=knowledge,
    ...
)
```

> 참고 예제: [`examples/functionality/rag/`](../examples/functionality/rag/)
> 모듈 문서: [rag](./rag.md) | [embedding](./embedding.md)

---

## STEP 7 — MCP 툴 연동

MCP 서버를 통해 외부 서비스를 툴로 사용한다.

```python
from agentscope.mcp import StdIOStatefulClient
from agentscope.tool import Toolkit

# Playwright로 브라우저 자동화
browser_client = StdIOStatefulClient(
    name="browser",
    command="npx",
    args=["@playwright/mcp@latest"],
)

# Tavily로 웹 검색
search_client = StdIOStatefulClient(
    name="search",
    command="npx",
    args=["tavily-mcp@latest"],
    env={"TAVILY_API_KEY": "YOUR_KEY"},
)

toolkit = Toolkit()

async with browser_client, search_client:
    await toolkit.register_mcp_client(browser_client)
    await toolkit.register_mcp_client(search_client)

    agent = ReActAgent(toolkit=toolkit, ...)
    response = await agent.reply(msg)
```

> 참고 예제: [`examples/functionality/mcp/`](../examples/functionality/mcp/)
> 모듈 문서: [mcp](./mcp.md)

---

## STEP 8 — 멀티 에이전트 워크플로우

여러 에이전트를 조합하여 복잡한 태스크를 처리한다.

### 순차 처리 (검토 → 수정 → 확인)

```python
from agentscope.pipeline import sequential_pipeline, MsgHub

writer = ReActAgent(name="작성자", sys_prompt="글을 작성합니다.", ...)
reviewer = ReActAgent(name="검토자", sys_prompt="글을 검토합니다.", ...)
editor = ReActAgent(name="편집자", sys_prompt="최종 편집을 합니다.", ...)

request = Msg(name="user", content="AI에 대한 블로그 글을 써주세요.", role="user")
final = await sequential_pipeline([writer, reviewer, editor], request)
```

### 브로드캐스트 (그룹 토론)

```python
async with MsgHub(
    participants=[agent_a, agent_b, agent_c],
    announcement=Msg(name="system", role="system", content="주제를 토론하세요."),
) as hub:
    for _ in range(3):  # 3라운드
        for agent in [agent_a, agent_b, agent_c]:
            msg = await agent.reply(last_msg)
            last_msg = msg
```

### 병렬 처리

```python
from agentscope.pipeline import fanout_pipeline

# 같은 질문을 여러 에이전트에게 동시에
results = await fanout_pipeline(
    agents=[expert_a, expert_b, expert_c],
    msg=question,
    sequential=False,  # 병렬 실행
)
```

> 참고 예제: [`examples/workflows/`](./examples/workflows.md)
> 모듈 문서: [pipeline](./pipeline.md)

---

## STEP 9 — 프로덕션 배포

HTTP API 서버로 에이전트를 서비스화한다.

```python
# pip install quart
from quart import Quart, request, Response
from agentscope.session import JsonSession
from agentscope.pipeline import stream_printing_messages

app = Quart(__name__)
session = JsonSession(save_dir="./sessions/")

@app.route("/chat", methods=["POST"])
async def chat():
    data = await request.get_json()
    user_id = data["user_id"]
    session_id = data["session_id"]
    message = data["message"]

    agent = create_agent()  # 에이전트 팩토리 함수
    await session.load_session_state(
        session_id=session_id, user_id=user_id,
        allow_not_exist=True, agent=agent,
    )

    user_msg = Msg(name="user", content=message, role="user")

    async def generate():
        async for msg, is_last in stream_printing_messages(
            agents=[agent],
            coroutine_task=agent.reply(user_msg),
        ):
            yield f"data: {msg.content}\n\n"
        await session.save_session_state(
            session_id=session_id, user_id=user_id, agent=agent,
        )

    return Response(generate(), mimetype="text/event-stream")

app.run(port=5000)
```

> 참고 예제: [`examples/deployment/planning_agent/`](./examples/deployment.md)
> 모듈 문서: [session](./session.md) | [pipeline](./pipeline.md)

---

## STEP 10 — 관찰성 및 디버깅

에이전트 동작을 추적하고 성능을 모니터링한다.

```python
from agentscope.tracing import setup_tracing

# Arize-Phoenix (로컬 UI)
import phoenix as px
px.launch_app()

setup_tracing(
    endpoint="http://localhost:6006/v1/traces",
    service_name="my-agent-app",
)

# 이후 모든 에이전트 호출이 자동으로 추적됨
agent = ReActAgent(...)
response = await agent.reply(msg)
# → localhost:6006 에서 스팬 트리 확인 가능
```

> 모듈 문서: [tracing](./tracing.md)

---

## 단계별 진행 체크리스트

```
□ STEP 1  — 설치 후 기본 에이전트 실행 성공
□ STEP 2  — 원하는 모델 제공자로 교체
□ STEP 3  — 커스텀 툴 1개 이상 추가
□ STEP 4  — 메모리로 멀티턴 대화 구현
□ STEP 5  — 세션 저장/복원으로 대화 유지
□ STEP 6  — 내 문서를 RAG로 연동
□ STEP 7  — MCP 서버 1개 이상 연동
□ STEP 8  — 멀티 에이전트 워크플로우 구현
□ STEP 9  — HTTP API 서버로 배포
□ STEP 10 — 추적/모니터링 연결
```

---

## 자주 쓰는 조합 예시

| 목적 | 조합 |
|------|------|
| 빠른 프로토타입 | `ReActAgent` + `InMemoryMemory` + 기본 툴 |
| 문서 Q&A 봇 | `ReActAgent` + `RAG` + `세션` |
| 웹 자동화 봇 | `ReActAgent` + `Playwright MCP` |
| 팀 토론 시뮬레이션 | `MsgHub` + 여러 `ReActAgent` |
| 음성 비서 | `RealtimeAgent` + `TTS` |
| 프로덕션 서비스 | `Quart` + `JsonSession` + `stream_printing` |
