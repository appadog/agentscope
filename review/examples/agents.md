# 에이전트 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/agent/`

8가지 에이전트 유형의 실제 구현 예제. 각 예제는 독립적으로 실행 가능하며 README를 포함한다.

---

## 1. react_agent — 기본 ReAct 에이전트

**파일**: `main.py`, `README.md`

**목적**: 툴을 사용하는 ReAct 에이전트의 기본 구현 예제.

```python
toolkit = Toolkit()
toolkit.register_tool_function(execute_shell_command)
toolkit.register_tool_function(execute_python_code)
toolkit.register_tool_function(view_text_file)

agent = ReActAgent(
    name="assistant",
    sys_prompt="...",
    model=DashScopeChatModel(model_name="qwen-max"),
    formatter=DashScopeChatFormatter(),
    toolkit=toolkit,
    memory=InMemoryMemory(),
)

# 대화 루프
while True:
    user_input = input("User: ")
    msg = Msg(name="user", content=user_input, role="user")
    response = await agent(msg)
```

**핵심 패턴**: 툴 등록 → ReAct 루프 → 스트리밍 응답

---

## 2. voice_agent — 음성 출력 에이전트

**파일**: `main.py`, `README.md`

**목적**: Qwen-Omni 모델을 사용하여 텍스트와 오디오를 동시에 생성하는 에이전트.

```python
agent = ReActAgent(
    name="voice_assistant",
    model=OpenAIChatModel(
        model_name="qwen-omni-turbo",
        base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    ),
    memory=InMemoryMemory(),
)
```

**특이사항**: 오디오 출력 모드가 활성화되면 툴 호출이 제한될 수 있음.

---

## 3. realtime_voice_agent — 실시간 음성 서버

**파일**: `run_server.py`, `chatbot.html`

**목적**: FastAPI + WebSocket으로 구성된 실시간 음성 대화 서버. 브라우저에서 접속 가능.

```python
@app.websocket("/ws/{user_id}/{session_id}")
async def websocket_endpoint(websocket: WebSocket, ...):
    await websocket.accept()

    # 에이전트 세션 생성 또는 복원
    agent = RealtimeAgent(
        name="assistant",
        realtime_model=DashScopeRealtimeModel(...),  # 또는 Gemini, OpenAI
    )

    # 클라이언트 이벤트 → 에이전트 → 오디오 응답
    outgoing_queue = asyncio.Queue()
    await agent.start(outgoing_queue)
```

**지원 모델**: `DashScopeRealtimeModel`, `GeminiRealtimeModel`, `OpenAIRealtimeModel`

**아키텍처**:
```
브라우저 (chatbot.html)
  ↕ WebSocket
FastAPI 서버 (run_server.py)
  ↕ 이벤트 큐
RealtimeAgent
  ↕ WebSocket
LLM 실시간 API
```

---

## 4. browser_agent — 브라우저 자동화 에이전트

**파일**: `main.py`, `browser_agent.py`, `README.md`

**목적**: Playwright MCP를 통해 웹 브라우저를 자동으로 조작하는 에이전트.

```python
mcp_client = StdIOStatefulClient(
    name="playwright",
    command="npx",
    args=["@playwright/mcp@latest"],
)

class BrowserAgent(ReActAgent):
    max_iterations: int = 20

    async def reply(self, msg: Msg, structured_model=None) -> Msg:
        # URL 추출 → 브라우저 실행 → 결과 반환
        ...

await mcp_client.connect()
toolkit = Toolkit()
await toolkit.register_mcp_client(mcp_client)
```

**핵심 패턴**: MCP를 통한 외부 툴(Playwright) 연동, 구조화 출력(Pydantic)으로 결과 검증

---

## 5. deep_research_agent — 딥리서치 에이전트

**파일**: `main.py`, 커스텀 `DeepResearchAgent` 클래스

**목적**: Tavily 검색 MCP와 연동하여 심층 리서치를 수행하는 에이전트.

```python
class DeepResearchAgent(ReActAgent):
    # 확장된 컨텍스트 처리
    # 중간 결과 파일 저장
    # Thinking 모드 활성화
    pass

search_client = StdIOStatefulClient(
    name="tavily",
    command="npx",
    args=["tavily-mcp@latest"],
)
```

**특징**:
- 긴 컨텍스트를 처리하기 위한 토큰 제한 확장
- 중간 연구 결과를 파일로 저장
- `thinking` 블록 지원 (Anthropic Claude 스타일)

---

## 6. meta_planner_agent — 계층적 플래닝 에이전트

**파일**: `main.py`, `tool.py`, `README.md`

**목적**: 복잡한 태스크를 서브태스크로 분해하고, 동적으로 워커 에이전트를 생성하여 실행하는 메타 플래너.

```python
# 플래너는 태스크를 직접 해결하지 않고 계획만 수립
sys_prompt = """
당신은 메타 플래너입니다.
- 태스크를 분석하여 서브태스크로 분해하세요.
- 각 서브태스크를 위한 워커 에이전트를 생성하세요.
- 서브태스크를 직접 해결하지 마세요.
"""

planner = ReActAgent(
    name="MetaPlanner",
    sys_prompt=sys_prompt,
    toolkit=toolkit,  # create_worker, PlanNotebook 툴 포함
    plan_notebook=PlanNotebook(),
)
```

**핵심 패턴**:
```
MetaPlanner.reply(태스크)
  → PlanNotebook.create_plan() → [SubTask1, SubTask2, ...]
  → create_worker(SubTask1) → WorkerAgent1.reply()
  → create_worker(SubTask2) → WorkerAgent2.reply()
  → 결과 통합
```

---

## 7. a2a_agent — Agent-to-Agent 클라이언트

**파일**: `main.py`, `agent_card.py`, `setup_a2a_server.py`, `README.md`

**목적**: A2A 프로토콜을 통해 원격 에이전트 서버와 통신하는 클라이언트 에이전트.

```python
agent = A2AAgent(
    name="client",
    resolver=FileAgentCardResolver("./agent_card.json"),
)

# 원격 에이전트에 태스크 전송
response = await agent.reply(
    Msg(name="user", content="파이썬으로 피보나치 수열을 구현해줘", role="user")
)
```

**제한사항**: 현재 채팅 전용, 스트리밍/구조화 출력 미지원

---

## 8. a2ui_agent — UI 생성 에이전트

**파일**: `__main__.py`, `skill/` 디렉토리

**목적**: 에이전트가 텍스트 대신 구조화된 UI 컴포넌트(폼, 카드, 리스트)로 응답하는 패턴.

```python
# 에이전트가 UI 스키마를 따르는 응답 생성
class ContactForm(BaseModel):
    name: str
    email: str
    message: str

response = await agent(msg, structured_model=ContactForm)
```

**지원 UI 컴포넌트**: 연락처 폼, 검색 필터, 선택 카드, 이미지 갤러리, 예약 폼

---

## 예제별 사용 모듈 매핑

| 예제 | Agent | Model | Tool | Memory | MCP | Special |
|------|-------|-------|------|--------|-----|---------|
| react_agent | ReActAgent | DashScope | ✓ | InMemory | — | — |
| voice_agent | ReActAgent | OpenAI | — | InMemory | — | Audio |
| realtime_voice | RealtimeAgent | Realtime | — | — | — | WebSocket |
| browser_agent | ReActAgent | Any | ✓ | — | Playwright | Pydantic |
| deep_research | ReActAgent | DashScope | ✓ | — | Tavily | Thinking |
| meta_planner | ReActAgent | Any | ✓ | — | — | PlanNotebook |
| a2a_agent | A2AAgent | — | — | — | — | A2A Protocol |
| a2ui_agent | Custom | Any | — | — | — | UI Schema |
