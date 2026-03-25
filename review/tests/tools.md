# 툴 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (5개)

| 파일 | 테스트 대상 |
|------|-----------|
| `tool_test.py` | 내장 툴 함수 (Python/Shell/파일) |
| `toolkit_basic_test.py` | Toolkit 핵심 동작 |
| `toolkit_meta_tool_test.py` | 메타 툴 기능 |
| `toolkit_middleware_test.py` | 툴 실행 미들웨어 |
| `tool_openai_test.py` / `tool_dashscope_test.py` | 모델별 툴 통합 |

---

## tool_test.py

내장 툴 함수들의 실행 결과 검증.

```python
class TestToolFunctions(IsolatedAsyncioTestCase):

    async def test_execute_python_code(self):
        """Python 코드 실행 및 출력 캡처"""
        result = await execute_python_code(
            code="print('hello world')\nx = 1 + 1\nprint(x)"
        )
        self.assertEqual(result.return_code, 0)
        self.assertIn("hello world", result.stdout)
        self.assertIn("2", result.stdout)

    async def test_python_code_error(self):
        """오류 발생 시 에러 메시지 반환"""
        result = await execute_python_code(
            code="raise ValueError('테스트 오류')"
        )
        self.assertNotEqual(result.return_code, 0)
        self.assertIn("ValueError", result.stderr)

    async def test_python_code_timeout(self):
        """무한 루프 코드 타임아웃 처리"""
        result = await execute_python_code(
            code="while True: pass",
            timeout=2,
        )
        self.assertNotEqual(result.return_code, 0)
        self.assertIn("timeout", result.stderr.lower())

    async def test_execute_shell_command(self):
        """셸 명령어 실행"""
        result = await execute_shell_command(command="echo hello")
        self.assertEqual(result.return_code, 0)
        self.assertIn("hello", result.stdout)

    async def test_view_text_file(self):
        """파일 내용 읽기 (범위 지정)"""
        # 임시 파일 생성
        with tempfile.NamedTemporaryFile(mode='w', suffix='.txt', delete=False) as f:
            f.write("line1\nline2\nline3\nline4\nline5\n")
            temp_path = f.name

        # 2~4번째 줄만 읽기
        result = await view_text_file(path=temp_path, start=2, end=4)
        self.assertIn("line2", result)
        self.assertIn("line3", result)
        self.assertNotIn("line1", result)

    async def test_tilde_expansion(self):
        """경로에서 ~ 자동 확장"""
        result = await view_text_file(path="~/nonexistent.txt")
        # ~ 가 실제 홈 디렉토리로 확장됐는지 검증

    async def test_write_text_file(self):
        """파일 쓰기"""
        temp_path = tempfile.mktemp(suffix='.txt')
        await write_text_file(path=temp_path, content="테스트 내용")

        with open(temp_path) as f:
            self.assertEqual(f.read(), "테스트 내용")
```

---

## toolkit_basic_test.py

Toolkit 클래스의 핵심 기능 검증.

```python
class TestToolkit(IsolatedAsyncioTestCase):

    async def test_register_sync_function(self):
        """동기 함수 등록 및 호출"""
        def add(a: int, b: int) -> int:
            return a + b

        toolkit = Toolkit()
        toolkit.register_tool_function(add)

        self.assertIn("add", toolkit.tools)

        response = await toolkit.call_tool(
            ToolUseBlock(id="1", name="add", input={"a": 1, "b": 2}, ...)
        )
        # 응답에 "3" 포함되어야 함

    async def test_register_async_function(self):
        """비동기 함수 등록 및 호출"""
        async def async_search(query: str) -> str:
            return f"{query}에 대한 검색 결과"

        toolkit = Toolkit()
        toolkit.register_tool_function(async_search)
        # 비동기 함수도 정상 동작해야 함

    async def test_register_generator_function(self):
        """비동기 제너레이터 함수 (스트리밍 툴)"""
        async def stream_data(n: int):
            for i in range(n):
                yield f"데이터 {i}"

        toolkit = Toolkit()
        toolkit.register_tool_function(stream_data)

        responses = []
        async for response in toolkit.call_tool(
            ToolUseBlock(id="1", name="stream_data", input={"n": 3}, ...)
        ):
            responses.append(response)

        self.assertEqual(len(responses), 3)

    async def test_tool_group_activation(self):
        """툴 그룹 활성화/비활성화"""
        toolkit = Toolkit()
        toolkit.register_tool_function(search, group_name="search_tools")
        toolkit.create_tool_group("search_tools", active=True)

        # 비활성화
        toolkit.update_tool_groups(["search_tools"], active=False)

        # 비활성화된 그룹의 툴은 호출 불가
        with self.assertRaises(ToolGroupInactiveError):
            await toolkit.call_tool(ToolUseBlock(name="search", ...))

    async def test_tool_execution_error_handling(self):
        """툴 실행 오류 시 ToolResponse에 에러 포함"""
        def failing_tool():
            raise RuntimeError("툴 실행 실패")

        toolkit.register_tool_function(failing_tool)
        response = await toolkit.call_tool(ToolUseBlock(name="failing_tool", ...))

        self.assertIn("RuntimeError", response.content[0].text)
```

---

## toolkit_middleware_test.py

미들웨어 체인 동작 검증.

```python
async def test_middleware_execution_order(self):
    """미들웨어 실행 순서 검증"""
    order = []

    def middleware_a(next_call):
        async def wrapper(tool_call):
            order.append("a_before")
            result = await next_call(tool_call)
            order.append("a_after")
            return result
        return wrapper

    def middleware_b(next_call):
        async def wrapper(tool_call):
            order.append("b_before")
            result = await next_call(tool_call)
            order.append("b_after")
            return result
        return wrapper

    toolkit.add_middleware(middleware_a)
    toolkit.add_middleware(middleware_b)

    await toolkit.call_tool(...)
    self.assertEqual(order, ["a_before", "b_before", "b_after", "a_after"])
```

---

## 검증 항목 요약

| 항목 | tool_test | toolkit_basic | middleware | openai/dashscope |
|------|----------|--------------|------------|-----------------|
| Python 코드 실행 | ✓ | — | — | — |
| 타임아웃 처리 | ✓ | — | — | — |
| 셸 명령어 | ✓ | — | — | — |
| 파일 읽기/쓰기 | ✓ | — | — | — |
| 동기/비동기 등록 | — | ✓ | — | — |
| 스트리밍 툴 | — | ✓ | — | — |
| 그룹 관리 | — | ✓ | — | — |
| 미들웨어 체인 | — | — | ✓ | — |
| 모델 통합 | — | — | — | ✓ |
