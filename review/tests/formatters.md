# 포매터 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (7개)

| 파일 | 테스트 대상 |
|------|-----------|
| `formatter_openai_test.py` | OpenAIChatFormatter |
| `formatter_anthropic_test.py` | AnthropicChatFormatter |
| `formatter_gemini_test.py` | GeminiChatFormatter |
| `formatter_dashscope_test.py` | DashScopeChatFormatter |
| `formatter_deepseek_test.py` | DeepSeekChatFormatter |
| `formatter_ollama_test.py` | OllamaChatFormatter |
| `formatter_a2a_test.py` | A2AFormatter |

---

## 공통 테스트 구조

각 포매터 테스트는 다음 케이스를 반복 검증:

```python
class TestXxxFormatter(IsolatedAsyncioTestCase):

    async def test_text_block(self):
        """단순 텍스트 메시지 변환"""
        msgs = [Msg(name="user", content="hello", role="user")]
        result = await formatter.format(msgs)
        self.assertEqual(result[0]["role"], "user")
        self.assertEqual(result[0]["content"], "hello")

    async def test_image_block(self):
        """이미지 블록 변환"""
        msg = Msg(
            name="user",
            role="user",
            content=[
                TextBlock(type="text", text="이 이미지를 설명해줘"),
                ImageBlock(type="image", source=URLSource(type="url", url="http://...")),
            ],
        )
        result = await formatter.format([msg])
        # 각 제공자 포맷 검증

    async def test_tool_use_block(self):
        """툴 호출 블록 변환"""

    async def test_tool_result_block(self):
        """툴 결과 블록 변환"""

    async def test_multi_agent_format(self):
        """멀티 에이전트 메시지 순서 및 역할 변환"""
```

---

## 제공자별 포맷 검증

### OpenAI 포매터

```python
async def test_openai_tool_calls(self):
    msg = Msg(
        name="assistant",
        role="assistant",
        content=[
            ToolUseBlock(
                type="tool_use",
                id="call_abc123",
                name="search",
                input={"query": "파이썬 튜토리얼"},
                raw_input='{"query": "파이썬 튜토리얼"}',
            )
        ],
    )
    result = await formatter.format([msg])

    # OpenAI 포맷 검증
    self.assertEqual(result[0]["role"], "assistant")
    self.assertIn("tool_calls", result[0])
    self.assertEqual(result[0]["tool_calls"][0]["type"], "function")
    self.assertEqual(result[0]["tool_calls"][0]["id"], "call_abc123")

async def test_openai_audio_block(self):
    """OpenAI audio 블록 → input_audio 포맷 변환"""
    # mock_audio_file.wav 파일로 테스트
    ...
```

### Anthropic 포매터

```python
async def test_anthropic_thinking_block(self):
    """Extended Thinking 블록 변환"""
    msg = Msg(
        name="assistant",
        role="assistant",
        content=[
            ThinkingBlock(type="thinking", thinking="이 문제를 단계별로..."),
            TextBlock(type="text", text="결론적으로..."),
        ],
    )
    result = await formatter.format([msg])

    # Anthropic 포맷에서 thinking 블록 유지 검증
    self.assertEqual(result[0]["content"][0]["type"], "thinking")

async def test_anthropic_image_source_types(self):
    """base64 및 URL 이미지 소스 처리"""
    ...
```

### Gemini 포매터

```python
async def test_gemini_role_mapping(self):
    """Gemini는 'user'/'model' 역할 사용 (assistant → model)"""
    msg = Msg(name="ai", content="응답", role="assistant")
    result = await formatter.format([msg])
    self.assertEqual(result[0]["role"], "model")  # "assistant" → "model"

async def test_gemini_parts_structure(self):
    """Gemini의 parts 배열 구조 검증"""
    result = await formatter.format([msg])
    self.assertIn("parts", result[0])
```

### DashScope 포매터

```python
async def test_dashscope_multimodal(self):
    """DashScope 멀티모달 메시지 처리"""
    # 이미지 + 텍스트 혼합 메시지
    ...
```

---

## 공통 검증 항목

| 항목 | OpenAI | Anthropic | Gemini | DashScope | DeepSeek | Ollama |
|------|--------|-----------|--------|-----------|----------|--------|
| TextBlock | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ImageBlock (URL) | ✓ | ✓ | ✓ | ✓ | — | ✓ |
| ImageBlock (Base64) | ✓ | ✓ | ✓ | ✓ | — | ✓ |
| AudioBlock | ✓ | ✓ | ✓ | ✓ | — | — |
| ToolUseBlock | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ToolResultBlock | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ThinkingBlock | — | ✓ | — | — | — | — |
| 멀티 에이전트 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
