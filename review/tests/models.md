# 모델 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (5개)

| 파일 | 테스트 클래스 |
|------|-------------|
| `model_openai_test.py` | `TestOpenAIChatModel` |
| `model_anthropic_test.py` | `TestAnthropicChatModel` |
| `model_gemini_test.py` | `TestGeminiChatModel` |
| `model_dashscope_test.py` | `TestDashScopeChatModel` |
| `model_ollama_test.py` | `TestOllamaChatModel` |

---

## model_openai_test.py

**테스트 클래스:** `TestOpenAIChatModel(IsolatedAsyncioTestCase)`

**목 응답 빌더:**

```python
def _create_mock_response(self, text="Hello") -> MagicMock:
    """기본 텍스트 응답 mock"""
    mock = MagicMock()
    mock.choices = [MagicMock()]
    mock.choices[0].message.content = text
    mock.choices[0].message.tool_calls = None
    mock.usage.prompt_tokens = 10
    mock.usage.completion_tokens = 5
    return mock

def _create_mock_response_with_tools(self, tool_name, tool_args) -> MagicMock:
    """툴 호출 응답 mock"""
    ...

def _create_mock_response_with_reasoning(self, thinking, text) -> MagicMock:
    """thinking 블록 포함 응답 mock"""
    ...

def _create_stream_mock(self, chunks: list[str]):
    """스트리밍 응답 async iterator mock"""
    async def mock_stream():
        for text in chunks:
            chunk = MagicMock()
            chunk.choices[0].delta.content = text
            chunk.choices[0].delta.tool_calls = None
            yield chunk
    return mock_stream()
```

**테스트 메서드:**

| 메서드 | 검증 내용 |
|--------|----------|
| `test_init_default_params()` | 기본 초기화 (AsyncOpenAI mock) |
| `test_init_with_custom_params()` | temperature, max_tokens, organization |
| `test_call_with_regular_model()` | 기본 텍스트 응답 파싱 |
| `test_call_with_tools_integration()` | ToolUseBlock 생성 검증 |
| `test_call_with_reasoning_effort()` | o3-mini의 reasoning_effort 파라미터 |
| `test_call_with_structured_model_integration()` | Pydantic `.parse()` API |
| `test_streaming_response_processing()` | 스트림 청크 집계 |
| `test_streaming_response_with_none_delta()` | None delta 처리 |

**스트리밍 테스트 패턴:**

```python
async def test_streaming_response_processing(self):
    model = OpenAIChatModel(model_name="gpt-4o", stream=True, api_key="test")

    with patch.object(model._client.chat.completions, "create") as mock_create:
        mock_create.return_value = self._create_stream_mock(["Hello", " World"])

        response = await model(messages=[{"role": "user", "content": "Hi"}])

    self.assertIsInstance(response, ChatResponse)
    self.assertEqual(response.content[0].text, "Hello World")
```

---

## model_anthropic_test.py

**테스트 클래스:** `TestAnthropicChatModel(IsolatedAsyncioTestCase)`

**목 클래스:**

```python
class AnthropicMessageMock:
    """Anthropic API 응답 메시지 mock"""
    def __init__(self, content_blocks, usage=None):
        self.content = content_blocks
        self.usage = usage or AnthropicUsageMock(input_tokens=50, output_tokens=20)

class AnthropicContentBlockMock:
    """Anthropic 콘텐츠 블록 (text, tool_use, thinking)"""
    def __init__(self, type, **kwargs):
        self.type = type
        for k, v in kwargs.items():
            setattr(self, k, v)

class AnthropicEventMock:
    """스트리밍 이벤트 mock"""
    def __init__(self, type, **kwargs):
        self.type = type
        for k, v in kwargs.items():
            setattr(self, k, v)
```

**테스트 메서드:**

| 메서드 | 검증 내용 |
|--------|----------|
| `test_init_default_params()` | 기본 max_tokens=2048, stream=True |
| `test_init_with_custom_params()` | thinking config, 커스텀 파라미터 |
| `test_call_with_regular_messages()` | 기본 메시지 처리 |
| `test_call_with_system_message()` | system 메시지 분리 |
| `test_call_with_thinking_enabled()` | ThinkingBlock 응답 파싱 |
| `test_call_with_tools_integration()` | 툴 정의 포맷 및 ToolUseBlock 파싱 |
| `test_streaming_response_processing()` | 이벤트 기반 스트리밍 |
| `test_generate_kwargs_integration()` | temperature, top_p, top_k |
| `test_call_with_structured_model_integration()` | 구조화 출력 (툴 강제) |

**스트리밍 이벤트 순서:**

```python
events = [
    AnthropicEventMock("message_start", message=AnthropicMessageMock([])),
    AnthropicEventMock("content_block_start", index=0, content_block=text_block),
    AnthropicEventMock("content_block_delta", index=0, delta=TextDeltaMock("Hello")),
    AnthropicEventMock("content_block_delta", index=0, delta=TextDeltaMock(" World")),
    AnthropicEventMock("content_block_stop", index=0),
    AnthropicEventMock("message_delta", usage=UsageMock(output_tokens=10)),
    AnthropicEventMock("message_stop"),
]
```

---

## model_gemini_test.py

**테스트 클래스:** `TestGeminiChatModel(IsolatedAsyncioTestCase)`

**목 클래스:**

```python
class GeminiResponseMock:
    """Gemini API 응답 mock"""
    def __init__(self, candidates, usage=None):
        self.candidates = candidates
        self.usage_metadata = usage or GeminiUsageMock(
            prompt_token_count=10,
            candidates_token_count=5,
        )

class GeminiFunctionCallMock:
    def __init__(self, id, name, args):
        self.id = id
        self.name = name
        self.args = args

class GeminiPartMock:
    def __init__(self, text=None, function_call=None, thought=None):
        self.text = text
        self.function_call = function_call
        self.thought = thought

class GeminiCandidateMock:
    def __init__(self, parts):
        self.content = MagicMock()
        self.content.parts = parts
```

**테스트 메서드:**

| 메서드 | 검증 내용 |
|--------|----------|
| `test_init_default_params()` | 기본 초기화 |
| `test_init_with_custom_params()` | thinking config, 커스텀 파라미터 |
| `test_call_with_regular_model()` | 기본 텍스트 생성 |
| `test_call_with_tools_integration()` | function_calling_config AUTO 모드 |
| `test_call_with_thinking_enabled()` | `thought=True` 파트 → ThinkingBlock |
| `test_call_with_structured_model_integration()` | JSON 스키마 강제 모드 |
| `test_streaming_response_processing()` | async generator 스트리밍 |
| `test_format_tools_with_nested_schema()` | `$ref` JSON 스키마 인라인 해결 |

**비동기 제너레이터 헬퍼:**

```python
async def _create_async_generator(self, items: list):
    """테스트용 async generator"""
    for item in items:
        yield item
```

**중첩 스키마 `$ref` 해결 테스트:**

```python
async def test_format_tools_with_nested_schema(self):
    """Pydantic 모델의 $ref 참조를 인라인으로 전개하는지 검증"""
    class Inner(BaseModel):
        value: int

    class Outer(BaseModel):
        inner: Inner
        name: str

    # $ref 없이 완전히 인라인 처리된 스키마로 변환되어야 함
    tools = model._format_tools([tool_with_pydantic_schema])
    schema = tools[0]["function_declarations"][0]["parameters"]
    self.assertNotIn("$ref", str(schema))
```

---

## 공통 검증 항목

| 항목 | OpenAI | Anthropic | Gemini | DashScope | Ollama |
|------|--------|-----------|--------|-----------|--------|
| 기본 텍스트 응답 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 스트리밍 응답 | ✓ | ✓ | ✓ | ✓ | ✓ |
| None delta 처리 | ✓ | — | — | — | — |
| 툴 호출 응답 파싱 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 토큰 사용량 추적 | ✓ | ✓ | ✓ | ✓ | — |
| ThinkingBlock 파싱 | ✓ | ✓ | ✓ | — | — |
| 구조화 출력 | ✓ | ✓ | ✓ | — | — |
| $ref 스키마 해결 | — | — | ✓ | — | — |
| system 메시지 분리 | — | ✓ | — | — | — |
| 커스텀 파라미터 | ✓ | ✓ | ✓ | ✓ | ✓ |
