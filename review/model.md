# Model 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

다양한 LLM 제공자(OpenAI, Anthropic, Gemini, DashScope, Ollama, Trinity)에 대한 API 호출을 추상화하는 모듈. 통일된 인터페이스와 응답 처리를 제공하며, 스트리밍 및 토큰 사용량 추적을 지원한다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_model_base.py` | 추상 기반 클래스 |
| `_model_response.py` | 응답 데이터 클래스 (콘텐츠 블록 포함) |
| `_model_usage.py` | 토큰 사용량 추적 |
| `_openai_model.py` | OpenAI API 래퍼 |
| `_anthropic_model.py` | Anthropic Claude 래퍼 |
| `_dashscope_model.py` | 알리바바 DashScope 래퍼 |
| `_ollama_model.py` | 로컬 Ollama 래퍼 |
| `_gemini_model.py` | Google Gemini 래퍼 |
| `_trinity_model.py` | Trinity 파인튜닝 모델 래퍼 |

---

## 핵심 클래스

### `ChatModelBase`

```python
class ChatModelBase:
    model_name: str
    stream: bool

    async def __call__(
        *args, **kwargs
    ) -> ChatResponse | AsyncGenerator[ChatResponse, None]

    def _validate_tool_choice(
        tool_choice: str,
        tools: list[dict]
    ) -> None
```

모든 모델의 추상 기반 클래스. `__call__`을 통해 동기/스트리밍 응답을 모두 처리.

---

### `ChatResponse`

```python
class ChatResponse(DictMixin):
    id: str
    created_at: datetime
    type: str
    content: Sequence[
        TextBlock | ToolUseBlock | ThinkingBlock | AudioBlock
    ]
    usage: ChatUsage
    metadata: dict
```

LLM 응답을 표현하는 데이터 클래스. 멀티모달 콘텐츠 블록 구조를 사용하여 텍스트, 툴 호출, 음성 응답을 통일된 포맷으로 처리.

---

### `ChatUsage`

```python
class ChatUsage:
    input_tokens: int
    output_tokens: int
    time: float        # 응답 시간 (초)
    metadata: dict     # 제공자별 추가 정보
```

---

## 지원 모델 제공자

| 제공자 | 클래스 | 특이사항 |
|--------|--------|----------|
| OpenAI | `OpenAIChatModel` | GPT-4o, o1 등 지원 |
| Anthropic | `AnthropicChatModel` | Claude 모델, extended thinking 지원 |
| Google | `GeminiChatModel` | Gemini 모델 |
| DashScope | `DashScopeChatModel` | Qwen 모델 (알리바바) |
| Ollama | `OllamaChatModel` | 로컬 LLM 실행 |
| Trinity | `TrinityChatModel` | 파인튜닝된 모델 서빙 |

---

## 실시간 모델

별도의 `RealtimeModelBase` 클래스 계층이 존재 ([realtime 모듈](./realtime.md) 참고):

```python
class RealtimeModelBase:
    support_input_modalities: list[str]   # ["audio", "text", "image"]
    websocket_url: str
    input_sample_rate: int
    output_sample_rate: int

    async def connect(outgoing_queue, instructions, tools)
    async def send(data: AudioBlock | TextBlock | ImageBlock | ToolResultBlock)
    async def disconnect()
```

---

## 모듈 연동

```
ChatModelBase
  ← Formatter   (Msg → API 요청 포맷)
  → ChatResponse (응답 반환)
  ← Agent       (모델 호출)
  → Tracing     (토큰 사용량 기록)
```

---

## 스트리밍 처리

```python
# 스트리밍 모드
async for chunk in model(messages, stream=True):
    # chunk: ChatResponse (부분 응답)
    process(chunk)

# 일반 모드
response: ChatResponse = await model(messages)
```
