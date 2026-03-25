# 기능 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/functionality/`

AgentScope의 개별 기능을 시연하는 13개 예제 모음.

---

## 1. rag — 검색 증강 생성

**파일**: `basic_usage.py`, `react_agent_integration.py`, `agentic_usage.py`, `multimodal_rag.py`

### 기본 사용

```python
# 임베딩 모델 + Qdrant 벡터 DB 설정
embedding_model = DashScopeTextEmbedding(model_name="text-embedding-v3")
store = QdrantStore(collection_name="docs", embedding_dim=1024)

knowledge = SimpleKnowledge(
    embedding_store=store,
    embedding_model=embedding_model,
)

# 문서 추가
reader = TextReader(chunk_size=512)
docs = await reader("./documents/")
await knowledge.add_documents(docs)

# 검색
results = await knowledge.retrieve("AgentScope란 무엇인가?", limit=5)
```

### 에이전트 통합

```python
agent = ReActAgent(
    name="rag_agent",
    knowledge_base=knowledge,  # 에이전트에 지식 베이스 연결
    ...
)
# 에이전트가 자동으로 retrieve_knowledge 툴을 사용
```

### 멀티모달 RAG

```python
mm_embedding = DashScopeMultimodalEmbedding(...)
# 이미지와 텍스트를 함께 임베딩하여 검색
```

---

## 2. short_term_memory — 단기 메모리 관리

### 메모리 압축 (MemoryWithCompress)

```python
agent = ReActAgent(
    name="assistant",
    memory=InMemoryMemory(),
    compression_config=ReActAgent.CompressionConfig(
        enable=True,
        agent_token_counter=CharTokenCounter(),
        trigger_threshold=2000,  # 토큰 임계값
        keep_recent=3,           # 최근 N개 메시지 보존
    ),
)
```

### ReMe 단기 메모리

```python
from agentscope.memory import ReMeShortTermMemory

memory = ReMeShortTermMemory(
    model=DashScopeChatModel(...),
    max_tokens=1500,
)
agent = ReActAgent(memory=memory, ...)
```

---

## 3. long_term_memory — 장기 메모리

### ReMe 개인 메모리

```python
from agentscope.memory import ReMePersonalLongTermMemory

long_term = ReMePersonalLongTermMemory(
    embedding_model=DashScopeTextEmbedding(...),
    storage_path="./personal_memory/",
)

agent = ReActAgent(
    long_term_memory=long_term,
    # 에이전트가 record_to_memory, retrieve_from_memory 툴 자동 사용
    ...
)
```

### Mem0 통합

```python
from agentscope.memory import Mem0LongTermMemory

long_term = Mem0LongTermMemory(
    user_id="user_001",
    config={"vector_store": {"provider": "qdrant", ...}},
)
```

---

## 4. structured_output — 구조화된 출력

```python
from pydantic import BaseModel

class PersonTable(BaseModel):
    name: str
    age: int
    occupation: str

class Choice(BaseModel):
    answer: Literal["A", "B", "C", "D"]
    reason: str

# 구조화 출력 요청
response = await agent(
    Msg(name="user", content="...", role="user"),
    structured_model=PersonTable,
)
# response.metadata에 검증된 Pydantic 객체 포함
```

---

## 5. plan — 플래닝

### 수동 계획 생성

```python
from agentscope.plan import PlanNotebook, SubTask

notebook = PlanNotebook()
plan = notebook.create_plan(
    name="research_task",
    subtasks=[
        SubTask(name="검색", description="웹에서 정보 수집"),
        SubTask(name="분석", description="수집된 정보 분석"),
        SubTask(name="보고서", description="분석 결과 정리"),
    ]
)
```

### 에이전트 자동 계획

```python
agent = ReActAgent(
    plan_notebook=PlanNotebook(),
    enable_meta_tool=True,  # 에이전트가 직접 계획 생성 가능
    ...
)
```

---

## 6. mcp — Model Context Protocol

```python
# HTTP SSE 클라이언트
sse_client = HttpStatefulClient(
    name="server1",
    url="http://localhost:8080/sse",
)

# Streamable HTTP 클라이언트 (무상태)
http_client = HttpStatelessClient(
    name="server2",
    url="http://localhost:8081/mcp",
)

toolkit = Toolkit()
await toolkit.register_mcp_client(sse_client)
await toolkit.register_mcp_client(http_client)

agent = ReActAgent(toolkit=toolkit, ...)
```

**동봉 MCP 서버 예제**:
- `mcp_add.py` — 덧셈 툴 서버
- `mcp_multiply.py` — 곱셈 툴 서버

---

## 7. tts — 텍스트-음성 변환

```python
from agentscope.tts import DashScopeRealtimeTTSModel

tts = DashScopeRealtimeTTSModel(
    model_name="cosyvoice-v1",
    voice="longyuan",
)

agent = ReActAgent(
    name="voice_agent",
    tts_model=tts,   # 응답을 자동으로 음성으로 변환
    ...
)
```

---

## 8. agent_skill — 에이전트 스킬

```python
# 스킬 디렉토리 구조
# skill/
#   ├── agentscope_overview.md
#   └── api_reference.md

toolkit = Toolkit()
toolkit.register_agent_skills("./skill/")
# 스킬이 에이전트의 지식으로 등록됨

agent = ReActAgent(toolkit=toolkit, ...)
```

---

## 9. stream_printing_messages — 스트리밍 출력

```python
from agentscope.pipeline import stream_printing_messages

# 에이전트 콘솔 직접 출력 비활성화
agent.set_console_output_enabled(False)

# 스트리밍으로 응답 처리
async for msg, is_last in stream_printing_messages(
    agents=[agent],
    coroutine_task=agent(user_msg),
):
    print(msg.content, end="", flush=True)
    if is_last:
        print()
```

---

## 10. session_with_sqlite — SQLite 세션

```python
from agentscope.session import SqliteSession

session = SqliteSession(db_path="./conversations.db")

# 멀티 유저 지원
await session.load_session_state(
    session_id="chat_001",
    user_id="alice",
    agent=agent,
    allow_not_exist=True,
)

# 대화 후 저장
await session.save_session_state(
    session_id="chat_001",
    user_id="alice",
    agent=agent,
)
```

---

## 11. vector_store — 벡터 DB 연동

다양한 벡터 DB 백엔드 예제:

| 벡터 DB | 디렉토리 | 특징 |
|---------|---------|------|
| MongoDB Atlas | `mongodb/` | 클라우드, 필터링 |
| Milvus Lite | `milvus_lite/` | 경량 임베디드 |
| OceanBase | `oceanbase/` | 알리바바 클라우드 |
| MySQL Vector | `alibabacloud_mysql_vector/` | SQL 기반 벡터 검색 |

```python
# Milvus Lite 예시
from agentscope.rag import MilvusLiteStore

store = MilvusLiteStore(
    uri="./milvus.db",
    collection_name="docs",
    embedding_dim=1024,
    metric_type="COSINE",
)
```

---

## 기능별 의존 모듈 매핑

| 예제 | 핵심 모듈 |
|------|----------|
| rag | RAG, Embedding, VDB |
| short_term_memory | Memory |
| long_term_memory | Memory, Embedding |
| structured_output | Agent, Message |
| plan | Plan, Agent |
| mcp | MCP, Tool |
| tts | TTS, Agent |
| agent_skill | Tool |
| stream_printing | Pipeline |
| session_with_sqlite | Session |
| vector_store | RAG, Embedding |
