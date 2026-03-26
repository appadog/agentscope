# 기능 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/functionality/`

---

## 1. rag — 검색 증강 생성

**파일**: `basic_usage.py`, `react_agent_integration.py`, `agentic_usage.py`, `multimodal_rag.py`

### basic_usage.py — 실제 코드

```python
# examples/functionality/rag/basic_usage.py
reader = TextReader(chunk_size=1024)
pdf_reader = PDFReader(chunk_size=1024, split_by="sentence")

# 텍스트 직접 파싱
documents = await reader(
    text="I'm Tony Stank, my password is 123456. My best friend is James Rhodes.",
)

# PDF 파일 파싱
pdf_documents = await pdf_reader(pdf_path=pdf_path)

# Qdrant 인메모리 + DashScope 임베딩
knowledge = SimpleKnowledge(
    embedding_store=QdrantStore(
        location=":memory:",
        collection_name="test_collection",
        dimensions=1024,
    ),
    embedding_model=DashScopeTextEmbedding(
        api_key=os.environ["DASHSCOPE_API_KEY"],
        model_name="text-embedding-v4",
    ),
)

await knowledge.add_documents(documents + pdf_documents)

# 유사도 검색
docs = await knowledge.retrieve(
    query="What is Tony Stank's password?",
    limit=3,
    score_threshold=0.7,
)
for doc in docs:
    print(f"Document ID: {doc.id}, Score: {doc.score}, Content: {doc.metadata.content['text']}")
```

**핵심 포인트:**
- `TextReader`는 파일 경로 대신 `text=` 파라미터로 직접 텍스트 파싱 가능
- `split_by="sentence"` — 문장 단위 청킹
- `dimensions=1024` — `text-embedding-v4` 모델의 벡터 차원

---

## 2. structured_output — 구조화 출력

**파일**: `main.py`

**실제 코드:**

```python
# examples/functionality/structured_output/main.py
class TableModel(BaseModel):
    name: str = Field(description="The name of the person")
    age: int = Field(description="The age of the person", ge=0, le=120)
    intro: str = Field(description="A one-sentence introduction")
    honors: list[str] = Field(description="A list of honors received")

class ChoiceModel(BaseModel):
    choice: Literal["apple", "banana", "orange"] = Field(
        description="Your choice of fruit",
    )

agent = ReActAgent(
    name="Friday",
    sys_prompt="You are a helpful assistant named Friday.",
    model=DashScopeChatModel(
        api_key=os.environ.get("DASHSCOPE_API_KEY"),
        model_name="qwen-max",
        stream=True,
    ),
    formatter=DashScopeChatFormatter(),
    toolkit=Toolkit(),
    memory=InMemoryMemory(),
)

# TableModel로 구조화 출력
res = await agent(Msg("user", "Please introduce Einstein", "user"),
                  structured_model=TableModel)
print(json.dumps(res.metadata, indent=4))

# Literal 타입으로 선택지 제한
res = await agent(Msg("user", "Choose one of your favorite fruit", "user"),
                  structured_model=ChoiceModel)
```

**결과:** `res.metadata`에 Pydantic 모델로 파싱된 딕셔너리 저장

---

## 3. mcp — MCP 툴 연동

**파일**: `main.py`, `mcp_add.py` (SSE 서버), `mcp_multiply.py` (StreamableHTTP 서버)

**실제 코드:**

```python
# examples/functionality/mcp/main.py
class NumberResult(BaseModel):
    result: int = Field(description="The result of the calculation")

toolkit = Toolkit()

# SSE 전송 방식 (stateful)
add_mcp_client = HttpStatefulClient(
    name="add_mcp",
    transport="sse",
    url="http://127.0.0.1:8001/sse",
)

# StreamableHTTP 전송 방식 (stateless)
multiply_mcp_client = HttpStatelessClient(
    name="multiply_mcp",
    transport="streamable_http",
    url="http://127.0.0.1:8002/mcp",
)

await add_mcp_client.connect()
await toolkit.register_mcp_client(add_mcp_client)
await toolkit.register_mcp_client(multiply_mcp_client)

agent = ReActAgent(
    name="Jarvis",
    sys_prompt="You're a helpful assistant named Jarvis.",
    model=DashScopeChatModel(model_name="qwen-max", ...),
    formatter=DashScopeChatFormatter(),
    toolkit=toolkit,
)

# 구조화 출력 + MCP 툴 함께 사용
res = await agent(
    Msg("user", "Calculate 2345 multiplied by 3456, then add 4567...", "user"),
    structured_model=NumberResult,
)

# MCP 툴을 로컬 callable로 직접 호출
add_tool_function = await add_mcp_client.get_callable_function(
    "add",
    wrap_tool_result=True,  # ToolResponse로 래핑
)
manual_res = await add_tool_function(a=5, b=10)

await add_mcp_client.close()  # stateful은 수동 close 필요
```

**핵심 포인트:**
- `HttpStatefulClient` vs `HttpStatelessClient` — 연결 유지 여부
- `get_callable_function()` — MCP 툴을 로컬 Python 함수처럼 직접 호출
- `wrap_tool_result=True` — 결과를 `ToolResponse` 객체로 래핑

---

## 4. session_with_sqlite — SQLite 세션 영속성

**파일**: `main.py`, `sqlite_session.py`

`SqliteSession`은 커스텀 구현 (AgentScope 내장 `JSONSession` 패턴을 SQLite로 확장).

**실제 코드:**

```python
# examples/functionality/session_with_sqlite/main.py
async def main(username: str, query: str) -> None:
    agent = ReActAgent(
        name="friday",
        sys_prompt="You are a helpful assistant named Friday.",
        model=DashScopeChatModel(model_name="qwen-max", ...),
        formatter=DashScopeChatFormatter(),
    )

    session = SqliteSession(SQLITE_PATH)

    # 이전 세션 복원 (없으면 새로 시작)
    await session.load_session_state(
        session_id=username,
        friday_of_user=agent,  # kwargs로 전달 = 상태 모듈 이름
    )

    await agent(Msg("user", query, "user"))

    # 상태 저장
    await session.save_session_state(
        session_id=username,
        friday_of_user=agent,
    )

# Alice와 Bob 각각 독립 세션
asyncio.run(main("alice", "What's the capital of America?"))
asyncio.run(main("bob", "What's the capital of China?"))

# Alice 세션 복원 후 이전 대화 확인
asyncio.run(main("alice", "What did I ask you before?"))
```

**핵심 포인트:** `session_id=username` + `kwargs`로 상태 모듈 이름 지정 — 멀티 에이전트도 동시 저장 가능

---

## 5. short_term_memory — 단기 메모리

**파일**: `memory_compression/main.py`, `reme/reme_short_term_memory.py`

### memory_compression/main.py

```python
from agentscope.memory import InMemoryMemory
from agentscope.agent import ReActAgent

agent = ReActAgent(
    name="assistant",
    memory=InMemoryMemory(),
    compression_config=ReActAgent.CompressionConfig(
        enable=True,
        trigger_threshold=3000,  # 3000 토큰 초과 시 압축
        keep_recent=5,           # 최근 5개 메시지 보존
    ),
    ...
)
```

---

## 6. long_term_memory — 장기 메모리

**파일**: `reme/`, `mem0/`

### ReMe 개인 장기 메모리

```python
# reme/personal_memory_example.py
from agentscope.memory import ReMePersonalLongTermMemory

memory = ReMePersonalLongTermMemory(
    api_key=os.environ["REME_API_KEY"],
    user_id="user_001",
)

# 대화 후 자동으로 개인 정보 추출 및 저장
await memory.add([user_msg, assistant_msg])

# 다음 대화에서 자동 검색
relevant_memories = await memory.retrieve(query="사용자 선호도")
```

### mem0 통합

```python
# mem0/memory_example.py
from agentscope.memory import Mem0LongTermMemory

memory = Mem0LongTermMemory(
    api_key=os.environ["MEM0_API_KEY"],
    user_id="user_001",
)
```

---

## 7. plan — 플랜 관리

**파일**: `main_agent_managed_plan.py`, `main_manual_plan.py`

```python
# main_agent_managed_plan.py (에이전트 자동 플랜 관리)
from agentscope.functionality.plan import PlanNotebook

notebook = PlanNotebook()
agent = ReActAgent(
    name="planner",
    plan_notebook=notebook,
    ...
)
# 에이전트가 자동으로 plan 생성/업데이트

# main_manual_plan.py (수동 플랜 관리)
plan = PlanNotebook()
await plan.add_task(SubTask(description="서브태스크 1"))
await plan.mark_completed(task_id)
```

---

## 8. tts — 텍스트 음성 합성

**파일**: `main.py`

```python
# examples/functionality/tts/main.py
from agentscope.tts import DashScopeTTSModel

tts = DashScopeTTSModel(
    model_name="qwen3-tts-flash",
    api_key=os.environ["DASHSCOPE_API_KEY"],
    voice="Cherry",
    stream=True,
)

agent = ReActAgent(
    name="voice_assistant",
    tts_model=tts,
    ...
)
```

---

## 9. stream_printing_messages — 스트리밍 출력

**파일**: `single_agent.py`, `multi_agent.py`

```python
# multi_agent.py
from agentscope.pipeline import stream_printing_messages

async for msg, is_last in stream_printing_messages(
    agents=[agent_a, agent_b],
    coroutine_task=agent_a(user_msg),
):
    print(f"[{msg.name}]: {msg.get_text_content()}")
```

---

## 10. vector_store — 벡터 스토어

**파일**: `milvus_lite/`, `mongodb/`, `alibabacloud_mysql_vector/`, `oceanbase/`

각 디렉토리에 해당 DB와 AgentScope RAG 연동 예제.

```python
# milvus_lite/main.py
from agentscope.rag import MilvusLiteStore

store = MilvusLiteStore(
    uri="./milvus_lite.db",
    collection_name="docs",
    dimensions=1024,
)
```

---

## 11. agent_skill — 에이전트 스킬

**파일**: `main.py`, `skill/`

Anthropic Agent Skills 패턴. 스킬을 Toolkit에 등록하여 에이전트가 활용.

```python
# main.py
from agentscope.tool import Toolkit

toolkit = Toolkit()
await toolkit.register_skill_from_directory("./skill/")
# 스킬 디렉토리의 모든 도구를 자동 등록
```
