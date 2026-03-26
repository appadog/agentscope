# 에이전트 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/agent/`

---

## 1. react_agent — 기본 ReAct 에이전트

**파일**: `main.py`

**실제 코드:**

```python
# examples/agent/react_agent/main.py
toolkit = Toolkit()
toolkit.register_tool_function(execute_shell_command)
toolkit.register_tool_function(execute_python_code)
toolkit.register_tool_function(view_text_file)

agent = ReActAgent(
    name="Friday",
    sys_prompt="You are a helpful assistant named Friday.",
    model=DashScopeChatModel(
        api_key=os.environ.get("DASHSCOPE_API_KEY"),
        model_name="qwen-max",
        enable_thinking=False,
        stream=True,
    ),
    formatter=DashScopeChatFormatter(),
    toolkit=toolkit,
    memory=InMemoryMemory(),
)

user = UserAgent("User")

msg = None
while True:
    msg = await user(msg)
    if msg.get_text_content() == "exit":
        break
    msg = await agent(msg)
```

**특징:**
- 에이전트 이름: `Friday`
- 내장 툴 3개: `execute_shell_command`, `execute_python_code`, `view_text_file`
- `UserAgent`로 대화 루프 관리
- `msg.get_text_content() == "exit"` 로 종료

---

## 2. browser_agent — 웹 브라우저 자동화

**파일**: `browser_agent.py`, `main.py`

`BrowserAgent`는 `ReActAgent`를 확장하며, Playwright MCP를 통해 실제 브라우저를 제어.

**실제 코드:**

```python
# examples/agent/browser_agent/main.py
class FinalResult(BaseModel):
    result: str = Field(description="The final result to the initial user query")


async def main(start_url_param="https://www.google.com", max_iters_param=50):
    toolkit = Toolkit()
    browser_client = StdIOStatefulClient(
        name="playwright-mcp",
        command="npx",
        args=["@playwright/mcp@latest"],
    )

    await browser_client.connect()
    await toolkit.register_mcp_client(browser_client)

    agent = BrowserAgent(
        name="Browser-Use Agent",
        model=DashScopeChatModel(
            api_key=os.environ.get("DASHSCOPE_API_KEY"),
            model_name="qwen3-max",
            stream=False,
        ),
        formatter=DashScopeChatFormatter(),
        memory=InMemoryMemory(),
        toolkit=toolkit,
        max_iters=max_iters_param,
        start_url=start_url_param,
    )
    user = UserAgent("User")

    msg = None
    while True:
        msg = await user(msg)
        if msg.get_text_content() == "exit":
            break
        msg = await agent(msg, structured_model=FinalResult)
        await agent.memory.clear()  # 매 태스크 후 메모리 초기화
```

**`BrowserAgent` 클래스 특징:**
- `ReActAgent`를 상속
- `token_counter=OpenAITokenCounter("gpt-4o")` — 토큰 카운팅
- `max_mem_length=20` — 메모리 최대 길이
- `start_url` — 브라우저 초기 URL
- 세 가지 전문 프롬프트: `pure_reasoning_prompt`, `observe_reasoning_prompt`, `task_decomposition_prompt`
- 매 태스크 후 `agent.memory.clear()` 호출 — 브라우저 상태만 유지

**실행 방법:**
```bash
python main.py --start-url https://www.google.com --max-iters 50
```

---

## 3. deep_research_agent — 딥 리서치

**파일**: `deep_research_agent.py`, `main.py`

`DeepResearchAgent`는 `ReActAgent`를 확장. Tavily 검색 MCP 클라이언트를 사용해 깊은 웹 리서치 수행.

**실제 코드:**

```python
# examples/agent/deep_research_agent/main.py
tavily_search_client = StdIOStatefulClient(
    name="tavily_mcp",
    command="npx",
    args=["-y", "tavily-mcp@latest"],
    env={"TAVILY_API_KEY": os.getenv("TAVILY_API_KEY", "")},
)

await tavily_search_client.connect()
agent = DeepResearchAgent(
    name="Friday",
    sys_prompt="You are a helpful assistant named Friday.",
    model=DashScopeChatModel(
        api_key=os.environ.get("DASHSCOPE_API_KEY"),
        model_name="qwen3-max",
        enable_thinking=False,
        stream=True,
    ),
    formatter=DashScopeChatFormatter(),
    memory=InMemoryMemory(),
    search_mcp_client=tavily_search_client,
    tmp_file_storage_dir=agent_working_dir,
    max_tool_results_words=10000,
)

msg = Msg("Bob", content=user_query, role="user")
result = await agent(msg)
```

**`DeepResearchAgent` 파라미터:**
- `search_mcp_client` — Tavily MCP 클라이언트 (필수)
- `max_iters=30` — 최대 반복 횟수
- `max_depth=3` — 재귀 리서치 최대 깊이
- `max_tool_results_words=10000` — 툴 결과 최대 단어 수
- `tmp_file_storage_dir` — 임시 파일 저장 경로

**내부 동작:** 질문을 서브쿼리로 분해 → 각 서브쿼리에 대해 Tavily 검색 → 결과를 파일로 저장 → 최종 종합 보고서 생성

---

## 4. a2a_agent — 에이전트 간 통신

**파일**: `main.py`, `setup_a2a_server.py`, `agent_card.py`

**파일 구조:**
- `setup_a2a_server.py` — A2A 서버 구동 (StarlettApplication)
- `agent_card.py` — AgentCard 정의 (이름, URL, capabilities)
- `main.py` — A2AAgent 클라이언트 예제

**A2A 서버 설정:**
```python
# setup_a2a_server.py
from agentscope.a2a import A2AStarletteApplication

app = A2AStarletteApplication(
    agent=react_agent,
    agent_card=agent_card,
)
# uvicorn으로 실행
```

---

## 5. a2ui_agent — UI 생성 에이전트

**파일**: `samples/general_agent/`

에이전트 응답에 구조화된 UI 컴포넌트를 포함. `---a2ui_JSON---` 구분자로 UI JSON 전달.

**UI 템플릿 예시** (`skills/A2UI_response_generator/UI_templete_examples/`):
- `contact_form.py` — 연락처 폼
- `email_compose_form.py` — 이메일 작성 폼
- `selection_card.py` — 선택 카드
- `item_detail_card_with_image.py` — 이미지 포함 상세 카드
- `booking_form.py` — 예약 폼

---

## 6. meta_planner_agent — 메타 플래너

**파일**: `main.py`, `tool.py`

복잡한 태스크를 하위 태스크로 분해하고, 각 하위 태스크를 처리할 worker 에이전트를 동적 생성.

**툴:**
```python
# tool.py
async def create_worker(
    description: str,
    system_prompt: str,
) -> str:
    """Create a worker agent to solve a specific task."""
    worker = ReActAgent(
        name="worker",
        sys_prompt=system_prompt,
        model=...,
        toolkit=Toolkit(),
    )
    result = await worker(Msg("user", description, "user"))
    return result.get_text_content()
```

---

## 7. realtime_voice_agent — 실시간 음성

**파일**: `run_server.py`

WebSocket 기반 실시간 양방향 음성 대화. `RealtimeAgent` 사용.

```python
# run_server.py (개요)
from agentscope.agent import RealtimeAgent
from agentscope.model import DashScopeRealtimeModel

agent = RealtimeAgent(
    name="assistant",
    model=DashScopeRealtimeModel(
        model_name="qwen-omni-turbo-realtime",
        api_key=os.environ["DASHSCOPE_API_KEY"],
    ),
)
# FastAPI + WebSocket으로 클라이언트와 연결
```

---

## 8. voice_agent — TTS 음성 에이전트

**파일**: `main.py`

텍스트 응답을 TTS로 변환하여 음성 출력.

```python
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
