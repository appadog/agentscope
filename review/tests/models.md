# 모델 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (5개)

| 파일 | 테스트 대상 |
|------|-----------|
| `model_openai_test.py` | OpenAIChatModel |
| `model_anthropic_test.py` | AnthropicChatModel |
| `model_gemini_test.py` | GeminiChatModel |
| `model_dashscope_test.py` | DashScopeChatModel |
| `model_ollama_test.py` | OllamaChatModel |

---

## 공통 테스트 패턴

```python
class TestOpenAIChatModel(IsolatedAsyncioTestCase):

    async def test_basic_call(self):
        """기본 API 호출 및 응답 파싱"""
        with patch("openai.AsyncOpenAI") as MockClient:
            mock = AsyncMock()
            MockClient.return_value = mock

            # 목 응답 설정
            mock.chat.completions.create.return_value = build_mock_response(
                text="안녕하세요"
            )

            model = OpenAIChatModel(model_name="gpt-4o-mini")
            response = await model(
                messages=[{"role": "user", "content": "hello"}]
            )

            self.assertIsInstance(response, ChatResponse)
            self.assertEqual(response.content[0].text, "안녕하세요")

    async def test_streaming_mode(self):
        """스트리밍 응답 청크 처리"""
        async def mock_stream():
            for text in ["안녕", "하세", "요"]:
                yield build_stream_chunk(text=text)

        mock.chat.completions.create.return_value = mock_stream()

        chunks = []
        async for chunk in model(messages, stream=True):
            chunks.append(chunk)

        full_text = "".join(c.content[0].text for c in chunks)
        self.assertEqual(full_text, "안녕하세요")
```

---

## model_openai_test.py

```python
class TestOpenAIChatModel(IsolatedAsyncioTestCase):

    async def test_custom_params(self):
        """커스텀 파라미터 전달 검증"""
        model = OpenAIChatModel(
            model_name="gpt-4o",
            temperature=0.7,
            max_tokens=1000,
        )
        # 파라미터가 API 호출에 포함되는지 검증
        ...

    async def test_structured_output(self):
        """response_format으로 JSON 스키마 강제"""
        class OutputSchema(BaseModel):
            answer: str
            confidence: float

        model = OpenAIChatModel(
            model_name="gpt-4o-mini",
        )
        response = await model(
            messages,
            response_format=OutputSchema,
        )
        # JSON 응답 파싱 검증
        ...

    async def test_reasoning_effort(self):
        """o1/o3 모델 reasoning_effort 파라미터"""
        model = OpenAIChatModel(model_name="o1-mini")
        response = await model(
            messages,
            reasoning_effort="high",
        )
        # ThinkingBlock이 응답에 포함되는지 검증
        ...

    async def test_tool_call_response(self):
        """모델이 tool_calls를 반환할 때 ToolUseBlock 변환"""
        mock_response = build_tool_call_response(
            tool_name="search",
            tool_args={"query": "파이썬"},
        )
        response = await model(messages)
        self.assertIsInstance(response.content[0], ToolUseBlock)
```

---

## model_anthropic_test.py

```python
async def test_extended_thinking(self):
    """Claude의 extended thinking 응답 처리"""
    mock_response = build_anthropic_response(
        thinking="이 문제는...",
        text="결론적으로...",
    )
    response = await model(messages)

    # ThinkingBlock + TextBlock 순서 검증
    self.assertIsInstance(response.content[0], ThinkingBlock)
    self.assertIsInstance(response.content[1], TextBlock)

async def test_usage_tracking(self):
    """토큰 사용량 정확히 파싱되는지 검증"""
    response = await model(messages)
    self.assertEqual(response.usage.input_tokens, 50)
    self.assertEqual(response.usage.output_tokens, 100)
```

---

## 공통 검증 항목

| 항목 | OpenAI | Anthropic | Gemini | DashScope | Ollama |
|------|--------|-----------|--------|-----------|--------|
| 기본 텍스트 응답 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 스트리밍 응답 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 툴 호출 응답 파싱 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 토큰 사용량 추적 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 구조화 출력 | ✓ | ✓ | ✓ | — | — |
| 커스텀 파라미터 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 에러 처리 | ✓ | ✓ | ✓ | ✓ | ✓ |
