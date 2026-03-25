# Tests 개요

> [← 요약으로 돌아가기](../summary.md)

## 구조

```
tests/
├── *_test.py           # 54개 테스트 파일
├── test.docx           # Word 테스트 파일
├── test.pptx           # PowerPoint 테스트 파일
└── test.xlsx           # Excel 테스트 파일
```

---

## 테스트 카테고리 요약

| 카테고리 | 파일 수 | 커버 범위 | 상세 |
|----------|---------|----------|------|
| [에이전트](./agents.md) | 5 | AgentBase, ReAct, A2A, 훅 | [→](./agents.md) |
| [포매터](./formatters.md) | 7 | OpenAI, Anthropic, Gemini, DashScope, DeepSeek, Ollama, A2A | [→](./formatters.md) |
| [모델](./models.md) | 5 | OpenAI, Anthropic, Gemini, DashScope, Ollama | [→](./models.md) |
| [메모리](./memory.md) | 4 | InMemory, SQL, Redis, ReMe, mem0 | [→](./memory.md) |
| [툴](./tools.md) | 5 | 코드 실행, 파일 조작, Toolkit | [→](./tools.md) |
| [RAG](./rag.md) | 3 | 문서 리더, 벡터 스토어, 지식 베이스 | [→](./rag.md) |
| [실시간](./realtime.md) | 4 | WebSocket, 이벤트, OpenAI/Gemini/DashScope | [→](./realtime.md) |
| [추적](./tracing.md) | 4 | 데코레이터, 변환, 추출, 유틸리티 | [→](./tracing.md) |
| [기타](./others.md) | 17 | TTS, MCP, 토큰, 세션, 파이프라인 등 | [→](./others.md) |

---

## 테스트 프레임워크

모든 테스트는 Python 표준 `unittest` 프레임워크 사용:

```python
import unittest

# 비동기 테스트
class MyTest(unittest.IsolatedAsyncioTestCase):
    async def asyncSetUp(self):
        # 비동기 초기화
        pass

    async def test_something(self):
        result = await some_async_function()
        self.assertEqual(result, expected)

    async def asyncTearDown(self):
        # 비동기 정리
        pass

# 동기 테스트
class MySyncTest(unittest.TestCase):
    def test_something(self):
        result = sync_function()
        self.assertEqual(result, expected)
```

---

## 공통 테스트 패턴

### 외부 의존성 목 처리

```python
from unittest.mock import AsyncMock, patch, MagicMock

# LLM API 목 처리
with patch("openai.AsyncOpenAI") as mock_openai:
    mock_client = AsyncMock()
    mock_openai.return_value = mock_client
    mock_client.chat.completions.create.return_value = mock_response
    ...
```

### 커스텀 목 클래스

```python
class MockChatModel(ChatModelBase):
    """테스트용 목 모델"""
    async def __call__(self, messages, **kwargs):
        return ChatResponse(
            content=[TextBlock(type="text", text="목 응답")],
            usage=ChatUsage(input_tokens=10, output_tokens=5),
        )

class MockFormatter(FormatterBase):
    async def format(self, msgs):
        return [{"role": m.role, "content": m.content} for m in msgs]
```

### 비동기 생성기 목

```python
async def mock_stream_response():
    for text in ["안", "녕", "하", "세", "요"]:
        yield ChatResponse(
            content=[TextBlock(type="text", text=text)],
        )

model.side_effect = mock_stream_response
```

### 파일 정리

```python
class TestFileOps(unittest.IsolatedAsyncioTestCase):
    def setUp(self):
        self.temp_dir = tempfile.mkdtemp()

    def tearDown(self):
        shutil.rmtree(self.temp_dir, ignore_errors=True)
```

---

## 테스트 실행

```bash
# 전체 테스트
python -m pytest tests/

# 특정 카테고리
python -m pytest tests/formatter_*_test.py

# 단일 파일
python -m pytest tests/react_agent_test.py -v

# 비동기 테스트만
python -m pytest tests/ -k "async"
```
