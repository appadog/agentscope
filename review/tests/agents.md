# 에이전트 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (5개)

| 파일 | 테스트 대상 |
|------|-----------|
| `react_agent_test.py` | ReActAgent 전체 동작, 훅, 구조화 출력 |
| `a2a_agent_test.py` | A2AAgent 원격 통신 |
| `a2a_resolver_test.py` | FileAgentCardResolver |
| `hook_test.py` | 에이전트 훅 시스템 |
| `user_input_test.py` | UserAgent 입력 처리 |

---

## react_agent_test.py

ReActAgent의 핵심 동작을 검증.

```python
class MockChatModel(ChatModelBase):
    """시나리오별 응답을 반환하는 목 모델"""
    responses: list[ChatResponse]
    call_count: int = 0

    async def __call__(self, messages, **kwargs):
        response = self.responses[self.call_count]
        self.call_count += 1
        return response

class TestReActAgent(IsolatedAsyncioTestCase):

    async def test_basic_reply(self):
        """기본 텍스트 응답 테스트"""
        model = MockChatModel(responses=[
            ChatResponse(content=[TextBlock(text="안녕하세요")]),
        ])
        agent = ReActAgent(name="test", model=model, ...)
        response = await agent.reply(Msg(name="user", content="hello"))
        self.assertIsInstance(response, Msg)
        self.assertIn("안녕하세요", response.content)

    async def test_tool_call_cycle(self):
        """툴 호출 → 결과 수신 → 최종 응답 사이클 테스트"""
        model = MockChatModel(responses=[
            # 1st call: 툴 호출 요청
            ChatResponse(content=[ToolUseBlock(id="call_1", name="search", ...)]),
            # 2nd call: 툴 결과 받은 후 최종 응답
            ChatResponse(content=[TextBlock(text="검색 결과에 따르면...")]),
        ])
        ...

    async def test_hooks_pre_post_reasoning(self):
        """ReAct 전용 reasoning 훅 테스트"""
        pre_called = []
        post_called = []

        async def pre_reasoning(agent, *args, **kwargs):
            pre_called.append(True)
            return args, kwargs

        async def post_reasoning(agent, response):
            post_called.append(True)
            return response

        agent.pre_reasoning.append(pre_reasoning)
        agent.post_reasoning.append(post_reasoning)

        await agent.reply(msg)

        self.assertTrue(pre_called)
        self.assertTrue(post_called)

    async def test_structured_output(self):
        """Pydantic 구조화 출력 검증"""
        class Answer(BaseModel):
            value: str
            confidence: float

        response = await agent.reply(msg, structured_model=Answer)
        self.assertIsInstance(response.metadata, Answer)
```

---

## hook_test.py

훅 시스템의 등록, 실행, 수정 동작 전반을 검증.

```python
class MyAgent(AgentBase):
    """훅 테스트용 에이전트"""
    records: list[str] = []

    async def reply(self, msg):
        self.records.append("reply_called")
        return Msg(name=self.name, content="응답", role="assistant")

class TestHookSystem(IsolatedAsyncioTestCase):

    async def test_pre_post_reply_hooks(self):
        """pre/post reply 훅 실행 순서 검증"""
        order = []

        async def pre_hook(agent, *args, **kwargs):
            order.append("pre")
            return args, kwargs

        async def post_hook(agent, response):
            order.append("post")
            return response

        agent = MyAgent(name="test", ...)
        agent.pre_reply.append(pre_hook)
        agent.post_reply.append(post_hook)

        await agent.reply(msg)
        self.assertEqual(order, ["pre", "post"])

    async def test_hook_modifies_output(self):
        """post_reply 훅이 응답 내용을 수정할 수 있는지 검증"""
        async def uppercase_hook(agent, response):
            response.content = response.content.upper()
            return response

        agent.post_reply.append(uppercase_hook)
        response = await agent.reply(msg)
        self.assertEqual(response.content, "응답".upper())

    async def test_hook_removal(self):
        """훅 제거 동작 검증"""
        agent.pre_reply.append(pre_hook)
        agent.pre_reply.remove(pre_hook)
        self.assertNotIn(pre_hook, agent.pre_reply)

    async def test_hook_clear(self):
        """전체 훅 초기화"""
        agent.pre_reply.extend([hook1, hook2, hook3])
        agent.pre_reply.clear()
        self.assertEqual(len(agent.pre_reply), 0)
```

---

## a2a_agent_test.py

원격 에이전트 통신 시뮬레이션 검증.

```python
class MockA2AClient:
    """원격 A2A 에이전트를 시뮬레이션하는 목 클라이언트"""
    async def send_message(self, message) -> TaskResult:
        return TaskResult(
            id="task_001",
            status=TaskStatus.COMPLETED,
            artifacts=[
                Artifact(
                    content=[TextPart(text="원격 에이전트의 응답")],
                )
            ],
        )

class TestA2AAgent(IsolatedAsyncioTestCase):

    async def test_reply_via_a2a_protocol(self):
        """A2A 프로토콜을 통한 원격 에이전트 응답 수신 테스트"""
        agent = A2AAgent(
            name="client",
            resolver=MockResolver(mock_client=MockA2AClient()),
        )
        response = await agent.reply(
            Msg(name="user", content="태스크를 처리해줘")
        )
        self.assertIn("원격 에이전트의 응답", response.content)

    async def test_artifact_content_extraction(self):
        """Artifact에서 콘텐츠 블록 추출 검증"""
        # 멀티모달 아티팩트 처리 테스트
        ...
```

---

## 테스트 커버리지 포인트

| 동작 | 테스트 여부 |
|------|------------|
| 기본 텍스트 응답 | ✓ |
| 툴 호출 사이클 (단일) | ✓ |
| 툴 호출 사이클 (병렬) | ✓ |
| 메모리 추가/조회 | ✓ |
| pre/post 훅 실행 순서 | ✓ |
| 훅의 출력 수정 | ✓ |
| 훅 제거/초기화 | ✓ |
| 구조화 출력 검증 | ✓ |
| A2A 프로토콜 통신 | ✓ |
| 에이전트 카드 로딩 | ✓ |
