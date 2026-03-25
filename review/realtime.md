# Realtime 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

WebSocket 기반 실시간 스트리밍 모델과의 상호작용을 담당하는 모듈. OpenAI, DashScope, Gemini의 실시간 API를 통해 음성 입출력 및 텍스트 스트리밍을 지원하며, RealtimeAgent 및 ChatRoom 파이프라인과 연동된다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_base.py` | 추상 실시간 모델 기반 클래스 |
| `_openai_realtime_model.py` | OpenAI Realtime API 구현 |
| `_dashscope_realtime_model.py` | DashScope 실시간 API 구현 |
| `_gemini_realtime_model.py` | Google Gemini Live API 구현 |
| `_events/` | 이벤트 타입 정의 |

---

## 핵심 클래스

### `RealtimeModelBase`

```python
class RealtimeModelBase:
    model_name: str
    support_input_modalities: list[str]   # ["audio", "text", "image"]
    websocket_url: str
    websocket_headers: dict

    # 오디오 설정
    input_sample_rate: int    # 입력 샘플레이트 (예: 16000)
    output_sample_rate: int   # 출력 샘플레이트 (예: 24000)

    async def connect(
        outgoing_queue: asyncio.Queue,
        instructions: str,
        tools: list[dict]
    ) -> None

    async def send(
        data: AudioBlock | TextBlock | ImageBlock | ToolResultBlock
    ) -> None

    async def disconnect() -> None
```

---

## 이벤트 시스템

실시간 모델은 이벤트 기반으로 동작:

```python
# 주요 이벤트 타입
class InputAudioBufferAppendEvent    # 오디오 입력 추가
class InputAudioBufferCommitEvent    # 오디오 입력 완료
class ResponseAudioDeltaEvent        # 오디오 응답 청크
class ResponseTextDeltaEvent         # 텍스트 응답 청크
class ResponseFunctionCallEvent      # 툴 호출 이벤트
class ResponseDoneEvent              # 응답 완료
class SessionUpdateEvent             # 세션 설정 업데이트
```

---

## 지원 제공자

| 제공자 | API | 입력 모달리티 | 출력 모달리티 |
|--------|-----|-------------|-------------|
| OpenAI | Realtime API | audio, text | audio, text |
| DashScope | 실시간 API | audio, text | audio, text |
| Gemini | Live API | audio, text, image | audio, text |

---

## RealtimeAgent 연동

```python
from agentscope.agent import RealtimeAgent
from agentscope.realtime import OpenAIRealtimeModel
from agentscope.tts import OpenAITTSModel

realtime_model = OpenAIRealtimeModel(
    model_name="gpt-4o-realtime-preview",
)

agent = RealtimeAgent(
    name="voice_assistant",
    realtime_model=realtime_model,
    instructions="당신은 친절한 음성 비서입니다.",
)
```

---

## ChatRoom 파이프라인

여러 실시간 에이전트를 하나의 채팅룸에서 운영:

```python
from agentscope.pipeline import ChatRoom

room = ChatRoom(agents=[agent1, agent2])

async with room:
    await room.start(outgoing_queue=audio_queue)
    # 이벤트가 라우팅되어 각 에이전트에 전달됨
```

---

## 오디오 처리 흐름

```
마이크 입력 (raw PCM)
  ↓
AudioBlock 생성 (base64 인코딩)
  ↓
RealtimeModel.send(AudioBlock)
  ↓
WebSocket → LLM 서버
  ↓
ResponseAudioDeltaEvent 수신
  ↓
outgoing_queue에 오디오 청크 추가
  ↓
스피커 출력
```

---

## 모듈 연동

```
Realtime
  ← Agent      (RealtimeAgent가 모델 사용)
  → Pipeline   (ChatRoom과 연동)
  → TTS        (텍스트 응답 음성 변환)
  → Message    (AudioBlock 처리)
```
