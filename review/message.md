# Message 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

AgentScope 내 모든 통신의 기본 단위인 `Msg` 클래스와 멀티모달 콘텐츠 블록을 정의하는 모듈. 텍스트, 이미지, 오디오, 비디오, 툴 호출 결과를 단일 메시지 구조에 담을 수 있다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_message_base.py` | `Msg` 클래스 정의 |
| `_message_block.py` | 콘텐츠 블록 타입 정의 |

---

## 핵심 클래스

### `Msg`

```python
class Msg:
    name: str                                  # 발신자 이름
    role: Literal["user", "assistant", "system"]
    content: str | Sequence[ContentBlock]      # 텍스트 또는 멀티모달 블록
    metadata: dict | None                      # 추가 메타데이터
    timestamp: datetime
    invocation_id: str | None                  # 툴 호출 추적 ID

    def to_dict() -> dict
    @classmethod
    def from_dict(json_data: dict) -> Msg
```

---

## 콘텐츠 블록 타입

모든 블록은 `TypedDict` 기반으로 정의되어 있다.

### 텍스트 블록

```python
class TextBlock(TypedDict):
    type: Literal["text"]
    text: str

class ThinkingBlock(TypedDict):
    type: Literal["thinking"]
    thinking: str    # Extended thinking (Anthropic Claude 전용)
```

### 미디어 블록

```python
class ImageBlock(TypedDict):
    type: Literal["image"]
    source: Base64Source | URLSource

class AudioBlock(TypedDict):
    type: Literal["audio"]
    source: Base64Source | URLSource

class VideoBlock(TypedDict):
    type: Literal["video"]
    source: Base64Source | URLSource
```

### 툴 블록

```python
class ToolUseBlock(TypedDict):
    type: Literal["tool_use"]
    id: str        # 툴 호출 고유 ID
    name: str      # 툴 함수명
    input: dict    # 파싱된 입력값
    raw_input: str # 원본 입력 (JSON 문자열)

class ToolResultBlock(TypedDict):
    type: Literal["tool_result"]
    id: str        # 대응하는 ToolUseBlock의 ID
    output: list[TextBlock | ImageBlock | AudioBlock | VideoBlock]
    name: str      # 툴 함수명
```

### 소스 타입

```python
class Base64Source(TypedDict):
    type: Literal["base64"]
    media_type: str   # "image/png", "audio/wav" 등
    data: str         # base64 인코딩 데이터

class URLSource(TypedDict):
    type: Literal["url"]
    url: str
```

---

## 사용 예시

```python
# 텍스트 메시지
msg = Msg(name="user", role="user", content="안녕하세요")

# 멀티모달 메시지
msg = Msg(
    name="user",
    role="user",
    content=[
        TextBlock(type="text", text="이 이미지를 분석해줘"),
        ImageBlock(type="image", source=URLSource(type="url", url="...")),
    ]
)

# 툴 호출 결과 메시지
msg = Msg(
    name="assistant",
    role="assistant",
    content=[
        ToolResultBlock(
            type="tool_result",
            id="call_123",
            name="search",
            output=[TextBlock(type="text", text="검색 결과...")],
        )
    ]
)
```

---

## 모듈 연동

```
Msg
  → Formatter   (API 제공자 포맷으로 변환)
  → Memory      (대화 이력 저장)
  → Pipeline    (에이전트 간 메시지 라우팅)
  → A2A         (원격 에이전트 전송)
```
