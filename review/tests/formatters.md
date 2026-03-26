# 포매터 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (7개)

| 파일 | 테스트 클래스 |
|------|-------------|
| `formatter_openai_test.py` | `TestOpenAIFormatter` |
| `formatter_anthropic_test.py` | `TestAnthropicChatFormatterFormatter` |
| `formatter_gemini_test.py` | `TestGeminiFormatter` |
| `formatter_dashscope_test.py` | `TestDashScopeFormatter` |
| `formatter_deepseek_test.py` | DeepSeekChatFormatter |
| `formatter_ollama_test.py` | OllamaChatFormatter |
| `formatter_a2a_test.py` | A2AFormatter |

---

## 공통 테스트 구조

각 포매터 테스트는 `setUp()`에서 메시지 픽스처를 만들고, 4가지 조합을 검증한다:

```python
class TestXxxFormatter(IsolatedAsyncioTestCase):

    def setUp(self):
        self.msgs_system = [Msg("system", "You're a helpful assistant.", "system")]
        self.msgs_conversation = [
            Msg("user", [TextBlock(type="text", text="안녕")], "user"),
            Msg("assistant", [TextBlock(type="text", text="반가워요")], "assistant"),
        ]
        self.msgs_tools = [
            Msg("assistant", [ToolUseBlock(type="tool_use", id="call_1", name="search", ...)], "assistant"),
            Msg("tool", [ToolResultBlock(type="tool_result", tool_use_id="call_1", ...)], "tool"),
        ]

    async def test_formatter(self):
        # 4가지 메시지 조합 테스트
        for msgs in [
            self.msgs_system + self.msgs_conversation + self.msgs_tools,
            self.msgs_conversation + self.msgs_tools,          # system 없음
            self.msgs_system + self.msgs_tools,                # conversation 없음
            self.msgs_system + self.msgs_conversation,         # tools 없음
        ]:
            result = await formatter.format(msgs)
            self.assertIsInstance(result, list)
```

---

## formatter_openai_test.py

**테스트 메서드:**
- `test_formatter()` — 4가지 조합 기본 포맷
- `test_formatter_with_extract_image_blocks()` — tool result의 이미지를 별도 메시지로 승격
- `test_multiagent_formatter()` — `OpenAIMultiAgentFormatter` 멀티에이전트 포맷
- `test_multiagent_formatter_with_promote_tool_result_images()` — 멀티에이전트 이미지 승격

**OpenAI 툴 호출 포맷:**

```python
# 입력 (AgentScope)
ToolUseBlock(
    type="tool_use",
    id="call_abc123",
    name="search",
    input={"query": "파이썬"},
    raw_input='{"query": "파이썬"}',
)

# 출력 (OpenAI API 형식)
{
    "role": "assistant",
    "content": None,
    "tool_calls": [{
        "type": "function",
        "id": "call_abc123",
        "function": {
            "name": "search",
            "arguments": '{"query": "파이썬"}'
        }
    }]
}
```

**이미지 승격 패턴:** tool result에 이미지 포함 시 별도 user 메시지로 추출하여 OpenAI vision 처리 지원.

---

## formatter_anthropic_test.py

**테스트 클래스:** `TestAnthropicChatFormatterFormatter`

**테스트 메서드:**
- `test_chat_formatter()` — 5가지 메시지 조합
- `test_multiagent_formatter()` — `AnthropicMultiAgentFormatter` 멀티에이전트

**Anthropic 특유 패턴:**

```python
# ThinkingBlock → Anthropic thinking 블록 유지
Msg("assistant", [
    ThinkingBlock(type="thinking", thinking="이 문제는 단계별로..."),
    TextBlock(type="text", text="결론적으로..."),
], "assistant")

# Anthropic 출력
{
    "role": "assistant",
    "content": [
        {"type": "thinking", "thinking": "이 문제는 단계별로..."},
        {"type": "text", "text": "결론적으로..."}
    ]
}
```

**툴 결과 포맷:**
```python
# tool_use_id로 매핑
{
    "role": "user",
    "content": [{
        "type": "tool_result",
        "tool_use_id": "call_1",
        "content": [{"type": "text", "text": "검색 결과"}]
    }]
}
```

---

## formatter_gemini_test.py

**테스트 메서드:**
- `test_chat_formatter()` — 이미지·오디오 포함 기본 포맷
- `test_chat_formatter_with_extract_image_blocks()` — 이미지 승격
- `test_multi_agent_formatter()` — `GeminiMultiAgentFormatter`
- `test_multi_agent_formatter_with_promote_tool_result_images()`

**Gemini 고유 규칙:**

```python
# assistant → "model" 역할 매핑
Msg("ai", "응답", "assistant")
# → {"role": "model", "parts": [{"text": "응답"}]}

# parts 기반 멀티모달 구조
{
    "role": "user",
    "parts": [
        {"text": "이미지를 분석해줘"},
        {"inline_data": {"mime_type": "image/png", "data": "base64..."}}
    ]
}

# 함수 호출
{
    "role": "model",
    "parts": [{
        "function_call": {
            "id": "call_001",
            "name": "search",
            "args": {"query": "파이썬"}
        }
    }]
}
```

---

## formatter_dashscope_test.py

**테스트 메서드:**
- `test_chat_formatter()` — 기본 포맷
- `test_chat_formatter_with_extract_media_blocks()` — 이미지/오디오/비디오 승격
- `test_multiagent_formatter()` — `DashScopeMultiAgentFormatter`
- `test_multiagent_formatter_with_promote_media_tool_result()` — 멀티에이전트 미디어 승격

**미디어 처리 패턴:**

```python
def _mock_save_base64_data(self, media_type: str, _base64_data: str) -> str:
    """base64 미디어 저장 경로 반환 mock"""
    if "audio" in media_type:
        return self.mock_audio_path  # "mock_audio.wav"
    elif "video" in media_type:
        return self.mock_video_path  # "mock_video.mp4"
    return self.mock_image_path     # "mock_image.png"
```

**지원 미디어 타입:** `ImageBlock(PNG)`, `AudioBlock(WAV, MP3)`, `VideoBlock(MP4)`

---

## 공통 검증 항목

| 항목 | OpenAI | Anthropic | Gemini | DashScope |
|------|--------|-----------|--------|-----------|
| TextBlock | ✓ | ✓ | ✓ | ✓ |
| ImageBlock (URL) | ✓ | ✓ | ✓ | ✓ |
| ImageBlock (Base64) | ✓ | ✓ | ✓ | ✓ |
| AudioBlock | ✓ | ✓ | ✓ | ✓ |
| VideoBlock | — | — | — | ✓ |
| ToolUseBlock | ✓ | ✓ | ✓ | ✓ |
| ToolResultBlock | ✓ | ✓ | ✓ | ✓ |
| ThinkingBlock | — | ✓ | — | — |
| 이미지 승격 | ✓ | — | ✓ | ✓ |
| 멀티에이전트 포매터 | ✓ | ✓ | ✓ | ✓ |
| assistant→model 역할 | — | — | ✓ | — |
