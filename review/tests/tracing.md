# 추적 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (4개)

| 파일 | 테스트 대상 |
|------|-----------|
| `tracing_test.py` | `@trace` 데코레이터 동작 |
| `tracing_converter_test.py` | 메시지 블록 → 스팬 속성 변환 |
| `tracing_extractor_test.py` | 스팬 속성 추출 |
| `tracing_utils_test.py` | 추적 유틸리티 함수 |

---

## tracing_test.py

```python
class TestTracingDecorator(IsolatedAsyncioTestCase):

    async def test_trace_async_function(self):
        """비동기 함수에 @trace 데코레이터 적용"""
        spans = []

        @trace_agent
        async def traced_reply(agent, msg):
            return Msg(name="assistant", content="응답", role="assistant")

        with trace_capture(spans):
            await traced_reply(agent, msg)

        self.assertEqual(len(spans), 1)
        self.assertEqual(spans[0].name, "agent.reply")

    async def test_trace_with_error(self):
        """에러 발생 시 스팬 상태 ERROR로 설정"""
        @trace_agent
        async def failing_reply(agent, msg):
            raise ValueError("테스트 에러")

        spans = []
        with trace_capture(spans), self.assertRaises(ValueError):
            await failing_reply(agent, msg)

        self.assertEqual(spans[0].status.status_code, StatusCode.ERROR)

    async def test_trace_disabled(self):
        """trace_enabled=False일 때 스팬 생성 안 됨"""
        trace_enabled.set(False)

        spans = []
        with trace_capture(spans):
            await traced_function(...)

        self.assertEqual(len(spans), 0)
        trace_enabled.set(True)

    async def test_trace_async_generator(self):
        """비동기 제너레이터 함수 추적"""
        @trace_llm
        async def streaming_model(messages):
            for text in ["a", "b", "c"]:
                yield ChatResponse(content=[TextBlock(text=text)])

        chunks = []
        async for chunk in streaming_model(messages):
            chunks.append(chunk)

        self.assertEqual(len(chunks), 3)
        # 스팬이 생성 완료 후 종료되어야 함

    async def test_token_usage_recorded(self):
        """토큰 사용량이 스팬 속성에 기록됨"""
        spans = []

        @trace_llm
        async def model_call(messages):
            return ChatResponse(
                content=[TextBlock(text="응답")],
                usage=ChatUsage(input_tokens=50, output_tokens=20),
            )

        with trace_capture(spans):
            await model_call(messages)

        attrs = spans[0].attributes
        self.assertEqual(attrs["llm.token.input"], 50)
        self.assertEqual(attrs["llm.token.output"], 20)
```

---

## tracing_converter_test.py

메시지 블록을 OpenTelemetry 스팬 속성으로 변환하는 로직 검증.

```python
class TestTracingConverter(unittest.TestCase):

    def test_text_block_conversion(self):
        """TextBlock → 스팬 속성"""
        block = TextBlock(type="text", text="안녕하세요")
        part = _convert_block_to_part(block)

        self.assertEqual(part["type"], "text")
        self.assertEqual(part["text"], "안녕하세요")

    def test_thinking_block_conversion(self):
        """ThinkingBlock (reasoning) → 스팬 속성"""
        block = ThinkingBlock(type="thinking", thinking="이 문제는...")
        part = _convert_block_to_part(block)

        self.assertEqual(part["type"], "thinking")
        self.assertEqual(part["thinking"], "이 문제는...")

    def test_tool_use_block_conversion(self):
        """ToolUseBlock → 스팬 속성"""
        block = ToolUseBlock(
            type="tool_use",
            id="call_001",
            name="search",
            input={"query": "파이썬"},
            raw_input='{"query": "파이썬"}',
        )
        part = _convert_block_to_part(block)

        self.assertEqual(part["type"], "tool_use")
        self.assertEqual(part["name"], "search")
        self.assertEqual(part["id"], "call_001")

    def test_tool_result_block_conversion(self):
        """ToolResultBlock → 스팬 속성"""
        ...

    def test_image_block_conversion(self):
        """ImageBlock → 스팬 속성 (URL 방식)"""
        block = ImageBlock(
            type="image",
            source=URLSource(type="url", url="https://example.com/img.png"),
        )
        part = _convert_block_to_part(block)
        self.assertEqual(part["source"]["url"], "https://example.com/img.png")
```

---

## 검증 항목 요약

| 항목 | tracing | converter | extractor | utils |
|------|---------|-----------|-----------|-------|
| 비동기 함수 추적 | ✓ | — | — | — |
| 동기 함수 추적 | ✓ | — | — | — |
| 제너레이터 추적 | ✓ | — | — | — |
| 에러 상태 기록 | ✓ | — | — | — |
| trace_enabled 플래그 | ✓ | — | — | — |
| 토큰 사용량 기록 | ✓ | — | ✓ | — |
| TextBlock 변환 | — | ✓ | — | — |
| ThinkingBlock 변환 | — | ✓ | — | — |
| ToolUseBlock 변환 | — | ✓ | — | — |
| ToolResultBlock 변환 | — | ✓ | — | — |
| ImageBlock 변환 | — | ✓ | — | — |
| 속성 추출 | — | — | ✓ | — |
| 유틸리티 함수 | — | — | — | ✓ |
