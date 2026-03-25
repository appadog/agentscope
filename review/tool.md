# Tool 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

에이전트가 외부 기능을 호출할 수 있도록 툴 등록, 실행, 라이프사이클 관리를 담당하는 모듈. MCP 클라이언트 연동, 에이전트 스킬 로딩, 미들웨어 체인을 통한 실행 파이프라인을 제공한다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_toolkit.py` | 핵심 `Toolkit` 클래스 |
| `_response.py` | 툴 실행 응답 구조 |
| `_types.py` | 타입 정의 |
| `_async_wrapper.py` | 동기/비동기 함수 래퍼 |
| `_coding/_python.py` | Python 코드 실행 툴 |
| `_coding/_shell.py` | Shell 명령어 실행 툴 |
| `_text_file/` | 파일 읽기/쓰기 툴 |
| `_multi_modality/` | 비전 및 멀티모달 툴 |

---

## 핵심 클래스

### `Toolkit`

```python
class Toolkit(StateModule):
    tools: dict[str, RegisteredToolFunction]   # 등록된 함수들
    groups: dict[str, ToolGroup]               # 그룹화된 툴
    skills: dict[str, AgentSkill]              # 에이전트 스킬
    _middlewares: list                          # 실행 미들웨어
```

**툴 등록**:
```python
toolkit.register_tool_function(
    tool_func=my_function,
    group_name="search",
    preset_kwargs={"timeout": 30},
    json_schema=None,  # 자동 추론 가능
)
```

**툴 그룹 관리**:
```python
toolkit.create_tool_group("search", description="검색 관련 툴", active=True)
toolkit.update_tool_groups(["search"], active=False)  # 비활성화
toolkit.remove_tool_groups(["search"])               # 삭제
```

**MCP 클라이언트 등록**:
```python
toolkit.register_mcp_client(mcp_client)
# MCP 서버의 모든 툴이 자동으로 toolkit에 등록됨
```

**에이전트 스킬 로딩**:
```python
toolkit.register_agent_skills(skill_dir="./skills/")
# 디렉토리의 스킬 파일을 자동 로드
```

**툴 실행**:
```python
async for response in toolkit.call_tool(tool_call: ToolUseBlock):
    # response: ToolResponse (스트리밍 지원)
    process(response)
```

---

### `ToolResponse`

```python
class ToolResponse:
    content: list[TextBlock | ImageBlock | AudioBlock | VideoBlock]
    metadata: dict
    stream: bool          # 스트리밍 여부
    is_last: bool         # 마지막 청크 여부
    is_interrupted: bool  # 실행 중단 여부
    id: str               # 툴 호출 ID
```

---

## 내장 툴

### Python 코드 실행

```python
from agentscope.tool import python_executor

# 코드를 샌드박스 환경에서 실행
result = await python_executor(code="print('hello')")
```

### Shell 명령어 실행

```python
from agentscope.tool import shell_executor

result = await shell_executor(command="ls -la")
```

### 파일 조작

```python
from agentscope.tool import read_text_file, write_text_file

content = await read_text_file(path="./data.txt")
await write_text_file(path="./output.txt", content="...")
```

---

## 미들웨어 패턴

툴 실행 파이프라인에 미들웨어를 추가하여 로깅, 인증, 레이트 리밋 등을 적용할 수 있다:

```
call_tool(ToolUseBlock)
  → middleware_1
    → middleware_2
      → 실제 함수 실행
    ← response
  ← response (가공된 응답)
```

---

## JSON Schema 자동 추론

Python 함수의 타입 힌트와 docstring에서 JSON Schema를 자동으로 생성:

```python
def search_web(query: str, max_results: int = 10) -> str:
    """웹에서 정보를 검색합니다.

    Args:
        query: 검색어
        max_results: 최대 결과 수
    """
    ...

# 자동 생성되는 schema:
# {
#   "name": "search_web",
#   "description": "웹에서 정보를 검색합니다.",
#   "parameters": {
#     "query": {"type": "string", "description": "검색어"},
#     "max_results": {"type": "integer", "default": 10}
#   }
# }
```

---

## 모듈 연동

```
Toolkit
  ← Agent      (툴 호출 요청)
  ← MCP        (MCP 서버 툴 등록)
  → Tracing    (툴 실행 추적)
  → Message    (ToolResultBlock 생성)
```
