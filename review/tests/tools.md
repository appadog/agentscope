# 툴 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (5개)

| 파일 | 테스트 클래스 |
|------|-------------|
| `tool_test.py` | 내장 툴 함수 (Python/Shell/파일) |
| `toolkit_basic_test.py` | `ToolkitBasicTest` |
| `toolkit_meta_tool_test.py` | 메타 툴 기능 |
| `toolkit_middleware_test.py` | 툴 실행 미들웨어 |
| `tool_openai_test.py` / `tool_dashscope_test.py` | 모델별 툴 통합 |

---

## toolkit_basic_test.py

**테스트 클래스:** `ToolkitBasicTest(IsolatedAsyncioTestCase)`

**테스트용 툴 함수들:**

```python
def sync_func(arg1: int, arg2: Optional[list[Union[str, int]]] = None) -> str:
    """동기 함수 툴"""
    return f"arg1={arg1}, arg2={arg2}"

async def async_func(raise_cancel: bool = False) -> str:
    """비동기 함수 툴"""
    if raise_cancel:
        raise asyncio.CancelledError("취소됨")
    return "async 완료"

async def async_generator_func(raise_cancel: bool = False):
    """스트리밍 비동기 제너레이터 툴"""
    for i in range(3):
        yield ToolResponse(
            content=[TextBlock(type="text", text=f"chunk_{i}")],
            stream=True,
            is_last=(i == 2),
        )
    if raise_cancel:
        raise asyncio.CancelledError("취소됨")

def sync_generator_func():
    """동기 제너레이터 툴"""
    for i in range(3):
        yield ToolResponse(
            content=[TextBlock(type="text", text=f"sync_chunk_{i}")],
            stream=True,
            is_last=(i == 2),
        )

async def async_func_return_async_generator(n: int):
    """비동기 함수가 비동기 제너레이터를 반환"""
    async def gen():
        for i in range(n):
            yield f"item_{i}"
    return gen()

async def async_func_return_sync_generator(n: int):
    """비동기 함수가 동기 제너레이터를 반환"""
    def gen():
        for i in range(n):
            yield f"item_{i}"
    return gen()
```

**구조화 모델 지원 테스트:**

```python
class StructuredModel(BaseModel):
    arg3: int

class MyBaseModel1(BaseModel):
    field_a: str

class MyBaseModel2(BaseModel):
    field_b: int

class ExtendedModelReusingBaseModel(MyBaseModel1, MyBaseModel2):
    """다중 상속 Pydantic 모델"""
    pass
```

**핵심 테스트 패턴:**

```python
# 동기 함수 등록 및 호출
toolkit = Toolkit()
toolkit.register_tool_function(sync_func)

response = await toolkit.call_tool(
    ToolUseBlock(id="1", name="sync_func", input={"arg1": 42}, type="tool_use",
                 raw_input='{"arg1": 42}')
)
self.assertIn("42", str(response))

# 스트리밍 제너레이터 툴
async for response in toolkit.call_tool(
    ToolUseBlock(id="2", name="async_generator_func",
                 input={"raise_cancel": False}, type="tool_use",
                 raw_input='{"raise_cancel": false}')
):
    self.assertIsInstance(response, ToolResponse)
    self.assertTrue(response.stream)
```

**취소 처리:**

```python
# asyncio.CancelledError 발생 시 응답에 에러 포함
response = await toolkit.call_tool(
    ToolUseBlock(..., input={"raise_cancel": True})
)
# CancelledError가 ToolResponse로 감싸져 반환됨
```

---

## toolkit_middleware_test.py

미들웨어 체인 실행 순서 검증.

```python
async def test_middleware_execution_order(self):
    order = []

    def middleware_a(next_call):
        async def wrapper(tool_call, **kwargs):
            order.append("a_before")
            result = await next_call(tool_call, **kwargs)
            order.append("a_after")
            return result
        return wrapper

    def middleware_b(next_call):
        async def wrapper(tool_call, **kwargs):
            order.append("b_before")
            result = await next_call(tool_call, **kwargs)
            order.append("b_after")
            return result
        return wrapper

    toolkit = Toolkit()
    toolkit.add_middleware(middleware_a)
    toolkit.add_middleware(middleware_b)
    toolkit.register_tool_function(simple_func)

    await toolkit.call_tool(tool_call)

    # 중첩 실행 순서: a_before → b_before → 실제 실행 → b_after → a_after
    self.assertEqual(order, ["a_before", "b_before", "b_after", "a_after"])
```

---

## tool_test.py

내장 툴 함수들의 실행 결과 검증.

```python
class TestToolFunctions(IsolatedAsyncioTestCase):

    async def test_execute_python_code(self):
        result = await execute_python_code(code="print('hello')\nprint(1+1)")
        self.assertEqual(result.return_code, 0)
        self.assertIn("hello", result.stdout)
        self.assertIn("2", result.stdout)

    async def test_python_code_error(self):
        result = await execute_python_code(code="raise ValueError('오류')")
        self.assertNotEqual(result.return_code, 0)
        self.assertIn("ValueError", result.stderr)

    async def test_python_code_timeout(self):
        result = await execute_python_code(
            code="while True: pass",
            timeout=2,
        )
        self.assertNotEqual(result.return_code, 0)

    async def test_execute_shell_command(self):
        result = await execute_shell_command(command="echo hello")
        self.assertEqual(result.return_code, 0)
        self.assertIn("hello", result.stdout)

    async def test_view_text_file_range(self):
        """범위 지정 파일 읽기"""
        with tempfile.NamedTemporaryFile(mode='w', suffix='.txt', delete=False) as f:
            f.write("line1\nline2\nline3\nline4\nline5\n")
            temp_path = f.name

        result = await view_text_file(path=temp_path, start=2, end=4)
        self.assertIn("line2", result)
        self.assertNotIn("line1", result)

    async def test_write_text_file(self):
        temp_path = tempfile.mktemp(suffix='.txt')
        await write_text_file(path=temp_path, content="테스트 내용")
        with open(temp_path) as f:
            self.assertEqual(f.read(), "테스트 내용")
```

---

## 검증 항목 요약

| 항목 | tool_test | toolkit_basic | middleware |
|------|----------|--------------|------------|
| Python 코드 실행 | ✓ | — | — |
| 타임아웃 처리 | ✓ | — | — |
| 셸 명령어 | ✓ | — | — |
| 파일 읽기/쓰기 | ✓ | — | — |
| 동기 함수 등록 | — | ✓ | — |
| 비동기 함수 등록 | — | ✓ | — |
| async 제너레이터 툴 | — | ✓ | — |
| sync 제너레이터 툴 | — | ✓ | — |
| CancelledError 처리 | — | ✓ | — |
| 다중 상속 Pydantic 모델 | — | ✓ | — |
| 미들웨어 실행 순서 | — | — | ✓ |
| 미들웨어 입출력 수정 | — | — | ✓ |
