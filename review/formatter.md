# Formatter 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

AgentScope의 `Msg` 객체를 각 LLM 제공자의 API 포맷으로 변환하는 레이어. 에이전트 로직과 모델 API 세부사항을 완전히 분리하여, 모델 제공자를 교체해도 에이전트 코드를 수정할 필요가 없다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_formatter_base.py` | 추상 기반 클래스 |
| `_truncated_formatter_base.py` | 메시지 자르기(truncation) 기능 추가 기반 클래스 |
| `_openai_formatter.py` | OpenAI API 포맷 변환 |
| `_anthropic_formatter.py` | Anthropic API 포맷 변환 |
| `_dashscope_formatter.py` | DashScope API 포맷 변환 |
| `_ollama_formatter.py` | Ollama API 포맷 변환 |
| `_gemini_formatter.py` | Google Gemini API 포맷 변환 |
| `_deepseek_formatter.py` | DeepSeek API 포맷 변환 |
| `_a2a_formatter.py` | Agent-to-Agent 프로토콜 포맷 변환 |

---

## 핵심 클래스

### `FormatterBase`

```python
class FormatterBase:
    async def format(
        *args, **kwargs
    ) -> list[dict[str, Any]]  # API 요청용 메시지 리스트

    def assert_list_of_msgs(msgs: list) -> None
    def convert_tool_result_to_string(output: list) -> str
```

---

### `TruncatedFormatterBase`

컨텍스트 윈도우 초과를 방지하기 위한 메시지 자르기 기능 제공.

```python
class TruncatedFormatterBase(FormatterBase):
    max_tokens: int | None
    truncation_strategy: str  # "oldest_first" 등

    def truncate(msgs: list[dict]) -> list[dict]
```

---

## 제공자별 변환 로직

### OpenAI

```python
# AgentScope Msg → OpenAI 메시지 포맷
{
    "role": "user",
    "content": [
        {"type": "text", "text": "..."},
        {"type": "image_url", "image_url": {"url": "data:image/...;base64,..."}},
    ]
}

# ToolUseBlock → OpenAI tool_calls
{
    "role": "assistant",
    "tool_calls": [
        {"id": "...", "type": "function", "function": {"name": "...", "arguments": "..."}}
    ]
}
```

### Anthropic

```python
# ThinkingBlock 지원 (Extended Thinking)
{
    "role": "assistant",
    "content": [
        {"type": "thinking", "thinking": "..."},
        {"type": "text", "text": "..."},
    ]
}
```

### Gemini

```python
# Gemini는 role이 "user"/"model"
{
    "role": "model",
    "parts": [{"text": "..."}]
}
```

---

## 멀티모달 처리

각 포매터는 `ImageBlock`, `AudioBlock`, `VideoBlock`을 해당 제공자의 포맷으로 변환:

| 블록 타입 | OpenAI | Anthropic | Gemini |
|-----------|--------|-----------|--------|
| ImageBlock | `image_url` | `image` source | `inline_data` |
| AudioBlock | `input_audio` | `document` | `inline_data` |
| VideoBlock | — | — | `file_data` |

로컬 파일(`file://`)의 경우 자동으로 base64 인코딩하여 처리.

---

## 모듈 연동

```
Msg (message 모듈)
  ↓
FormatterBase.format()
  ↓
list[dict]  (API 요청 메시지)
  ↓
ChatModelBase.__call__()  (model 모듈)
```
