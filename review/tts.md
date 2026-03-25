# TTS 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

텍스트-음성(Text-to-Speech) 합성을 담당하는 모듈. OpenAI, Gemini, DashScope(CosyVoice 포함)의 TTS API를 지원하며, 실시간 스트리밍 합성과 단일 요청 합성을 모두 제공한다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_tts_base.py` | 추상 TTS 기반 클래스 |
| `_tts_response.py` | TTS 응답 구조 |
| `_openai_tts_model.py` | OpenAI TTS API |
| `_dashscope_cosyvoice_tts_model.py` | DashScope CosyVoice |
| `_dashscope_realtime_tts_model.py` | DashScope 실시간 TTS |
| `_gemini_tts_model.py` | Google Gemini TTS |
| `_utils.py` | 오디오 처리 유틸리티 |

---

## 핵심 클래스

### `TTSModelBase`

```python
class TTSModelBase:
    model_name: str
    stream: bool
    supports_streaming_input: bool   # 스트리밍 텍스트 입력 지원 여부

    # 컨텍스트 매니저 (스트리밍 모델용)
    async def __aenter__() -> TTSModelBase
    async def __aexit__(...) -> None

    async def connect() -> None     # 스트리밍 세션 시작
    async def close() -> None       # 세션 종료

    # 스트리밍: 텍스트 청크를 순차적으로 전송
    async def push(msg: Msg) -> TTSResponse

    # 단일 요청: 전체 텍스트를 한 번에 합성
    async def synthesize(msg: Msg) -> TTSResponse
```

---

### `TTSResponse`

```python
class TTSResponse:
    audio: AudioBlock          # 합성된 오디오
    metadata: dict             # 지속 시간, 샘플레이트 등
    is_last: bool              # 스트리밍 마지막 청크 여부
```

---

## 지원 제공자

| 제공자 | 모델 | 스트리밍 | 특징 |
|--------|------|----------|------|
| OpenAI | `tts-1`, `tts-1-hd` | 지원 | 6가지 음성 옵션 |
| DashScope CosyVoice | `cosyvoice-v1` | 지원 | 다국어, 감정 제어 |
| DashScope Realtime | 실시간 모델 | 실시간 | 초저지연 |
| Gemini | `gemini-2.5-flash` | 지원 | 멀티스피커 지원 |

---

## 사용 패턴

### 단일 요청 합성

```python
tts = OpenAITTSModel(model_name="tts-1", voice="alloy")

response = await tts.synthesize(
    Msg(name="assistant", role="assistant", content="안녕하세요!")
)

audio_data = response.audio  # AudioBlock
```

### 스트리밍 합성

```python
tts = DashScopeCosyVoiceTTSModel(model_name="cosyvoice-v1")

async with tts:
    # LLM이 텍스트를 생성할 때마다 순차적으로 전송
    async for text_chunk in llm_stream:
        response = await tts.push(
            Msg(name="assistant", role="assistant", content=text_chunk)
        )
        if response:
            play_audio(response.audio)
```

### RealtimeAgent 연동

```python
agent = RealtimeAgent(
    name="voice_bot",
    realtime_model=realtime_model,
    tts_model=tts_model,    # 텍스트 응답을 음성으로 변환
)
```

---

## 오디오 출력 형식

| 제공자 | 형식 | 샘플레이트 |
|--------|------|----------|
| OpenAI | MP3, Opus, AAC | 가변 |
| DashScope | PCM, MP3 | 22050/44100 Hz |
| Gemini | PCM | 24000 Hz |

---

## 모듈 연동

```
TTS
  ← Agent      (RealtimeAgent에서 텍스트 → 음성 변환)
  → Message    (AudioBlock 생성)
  ← Pipeline   (ChatRoom에서 음성 출력)
```
