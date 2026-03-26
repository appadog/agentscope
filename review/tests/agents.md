# 에이전트 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (5개)

| 파일 | 테스트 대상 |
|------|-----------|
| `react_agent_test.py` | ReActAgent 훅, 툴 호출 사이클 |
| `a2a_agent_test.py` | A2AAgent 원격 통신, 메시지 병합 |
| `a2a_resolver_test.py` | FileAgentCardResolver |
| `hook_test.py` | pre/post 훅 시스템 전반 |
| `user_input_test.py` | UserAgent 입력 처리 |

---

## react_agent_test.py

**테스트 클래스:** `ReActAgentTest(IsolatedAsyncioTestCase)`

실제 사용하는 목 모델:

```python
class MyModel(ChatModelBase):
    """시나리오별 응답을 순서대로 반환하는 목 모델"""
    def __init__(self) -> None:
        super().__init__("test_model", stream=False)
        self.cnt = 1
        self.fake_content_1 = [TextBlock(type="text", text="123")]
        self.fake_content_2 = [
            TextBlock(type="text", text="456"),
            ToolUseBlock(
                type="tool_use",
                name="generate_response",
                id="xx",
                input={"result": "789"},
            ),
        ]

    async def __call__(self, _messages: list[dict], **kwargs: Any) -> ChatResponse:
        self.cnt += 1
        if self.cnt == 2:
            return ChatResponse(content=self.fake_content_1)
        else:
            return ChatResponse(content=self.fake_content_2)
```

**핵심 테스트:** `test_react_agent()`

```python
async def test_react_agent(self):
    """ReActAgent 훅 등록 → 실행 → 카운터 검증"""
    pre_reasoning_cnt = 0
    post_reasoning_cnt = 0
    pre_acting_cnt = 0
    post_acting_cnt = 0

    async def pre_reasoning_hook(agent, *args, **kwargs):
        nonlocal pre_reasoning_cnt
        pre_reasoning_cnt += 1
        return args, kwargs

    async def post_reasoning_hook(agent, response):
        nonlocal post_reasoning_cnt
        post_reasoning_cnt += 1
        return response

    # ... pre/post acting 훅도 동일 패턴

    agent = ReActAgent(
        name="test",
        model=MyModel(),
        formatter=MockFormatter(),
        toolkit=toolkit_with_generate_response,
    )

    agent.pre_reasoning.append(pre_reasoning_hook)
    agent.post_reasoning.append(post_reasoning_hook)
    agent.pre_acting.append(pre_acting_hook)
    agent.post_acting.append(post_acting_hook)

    response = await agent.reply(Msg(name="user", content="hello", role="user"))

    # 툴 호출 1회 → reasoning 2회, acting 1회
    self.assertEqual(pre_reasoning_cnt, 2)
    self.assertEqual(post_reasoning_cnt, 2)
    self.assertEqual(pre_acting_cnt, 1)
    self.assertEqual(post_acting_cnt, 1)
```

**테스트 툴:** `generate_response(result: str)` — ReActAgent에 최종 응답을 전달하는 특수 내장 툴

---

## hook_test.py

**테스트 클래스:** `HookTest(IsolatedAsyncioTestCase)`

**에이전트 계층 구조:**

```python
class MyAgent(AgentBase):
    async def reply(self, msg, **kwargs):
        return Msg(name=self.name, content="reply", role="assistant")

    async def observe(self, msgs):
        pass

    async def handle_interrupt(self, msg):
        pass

class ChildAgent(MyAgent): pass
class GrandChildAgent(ChildAgent): pass
class AgentA(AgentBase): ...
class AgentB(AgentBase): ...
class AgentC(AgentBase): ...
```

**8가지 훅 함수 유형:**

```python
# 1. 비동기 pre-훅 (kwargs 수정)
async def async_pre_func_w_modifying(agent, *args, **kwargs):
    kwargs["extra"] = "added"
    return args, kwargs

# 2. 비동기 pre-훅 (수정 없음)
async def async_pre_func_wo_modifying(agent, *args, **kwargs):
    return args, kwargs

# 3. 동기 pre-훅 (kwargs 수정)
def sync_pre_func_w_modifying(agent, *args, **kwargs):
    kwargs["extra"] = "added"
    return args, kwargs

# 4-8: async/sync post-훅 variants (출력 Msg 수정)
async def async_post_func_w_modifying(agent, response: Msg) -> Msg:
    response.content = "modified"
    return response
```

**테스트 메서드:**
- `test_reply_hooks()` — pre/post reply 훅 실행 순서 및 내용 수정 검증
- `test_print_hooks()` — pre_print 훅 실행 검증
- `test_observe_hooks()` — pre/post observe 훅 실행 검증

---

## a2a_agent_test.py

**테스트 클래스:** `A2AAgentTest(IsolatedAsyncioTestCase)`

**목 클라이언트:**

```python
class MockA2AClient:
    """원격 A2A 에이전트 시뮬레이션"""
    def __init__(self, response_type="message"):
        self.response_type = response_type

    async def send_message(self, message) -> dict:
        if self.response_type == "task":
            return {
                "type": "task",
                "id": "task_001",
                "status": {"state": "completed"},
                "artifacts": [{"parts": [{"type": "text", "text": "작업 완료"}]}],
            }
        elif self.response_type == "message":
            return {
                "type": "message",
                "parts": [{"type": "text", "text": "에이전트 응답"}],
            }
        else:
            raise ValueError("에러 응답")

class MockClientFactory:
    """AgentCard → MockA2AClient 생성 팩토리"""
    def __init__(self, response_type="message"):
        self.response_type = response_type

    async def __call__(self, agent_card: AgentCard):
        return MockA2AClient(self.response_type)
```

**에이전트 카드 설정:**

```python
self.test_agent_card = AgentCard(
    name="TestAgent",
    url="http://localhost:8000",
    description="Test A2A agent",
    version="1.0.0",
    capabilities=AgentCapabilities(),
    default_input_modes=["text/plain"],
    default_output_modes=["text/plain"],
    skills=[],
)
self.agent = A2AAgent(self.test_agent_card)
```

**테스트 메서드:**

| 메서드 | 검증 내용 |
|--------|----------|
| `test_reply_with_task()` | task 응답 수신 및 artifacts 파싱 |
| `test_reply_with_no_messages()` | None/빈 리스트/None 요소 처리 |
| `test_observe_method()` | 메시지 관찰 기능 |
| `test_observe_and_reply_merge()` | 관찰된 메시지 + 입력 메시지 병합 |
| `test_reply_with_only_observed_messages()` | 관찰된 메시지만으로 응답 |

**핵심 동작:** reply 후 `_observed_messages` 자동 초기화

---

## 테스트 커버리지

| 동작 | react | a2a | hook |
|------|-------|-----|------|
| 기본 텍스트 응답 | ✓ | ✓ | — |
| 툴 호출 사이클 | ✓ | — | — |
| reasoning 훅 (ReAct 전용) | ✓ | — | — |
| acting 훅 (ReAct 전용) | ✓ | — | — |
| pre/post reply 훅 | — | — | ✓ |
| pre/post observe 훅 | — | — | ✓ |
| pre_print 훅 | — | — | ✓ |
| 동기/비동기 훅 혼용 | — | — | ✓ |
| 훅 출력 수정 | — | — | ✓ |
| A2A task 응답 | — | ✓ | — |
| A2A message 응답 | — | ✓ | — |
| 메시지 관찰/병합 | — | ✓ | — |
| 훅 상속 | — | — | ✓ |
