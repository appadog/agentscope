# MCP 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

Model Context Protocol(MCP) 서버와의 통합을 담당하는 모듈. stdio 및 HTTP 전송 방식을 지원하며, MCP 서버의 툴을 AgentScope의 Toolkit에 자동으로 등록할 수 있다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_client_base.py` | 추상 MCP 클라이언트 |
| `_stateful_client_base.py` | 세션 기반 클라이언트 기반 클래스 |
| `_http_stateful_client.py` | HTTP(SSE) 전송 |
| `_stdio_stateful_client.py` | stdio 전송 |
| `_http_stateless_client.py` | 상태 없는 HTTP 클라이언트 |
| `_mcp_function.py` | MCP 툴을 AgentScope 함수로 래핑 |

---

## 핵심 클래스

### `MCPClientBase`

```python
class MCPClientBase:
    name: str

    async def get_callable_function(
        func_name: str,
        wrap_tool_result: bool = True
    ) -> Callable

    def _convert_mcp_content_to_as_blocks(
        mcp_content_blocks: list
    ) -> list[TextBlock | ImageBlock | AudioBlock]
```

MCP 서버 응답을 AgentScope의 콘텐츠 블록으로 변환하는 핵심 메서드 포함.

---

### `StatefulClientBase`

```python
class StatefulClientBase(MCPClientBase):
    is_connected: bool
    client: Any
    stack: AsyncExitStack
    session: ClientSession

    async def connect() -> None
    async def close() -> None

    async def list_tools() -> list[mcp.types.Tool]

    async def get_callable_function(
        func_name: str,
        wrap_tool_result: bool = True
    ) -> Callable
```

지속적인 세션을 유지하며 MCP 서버와 통신.

---

## 전송 방식

### stdio 클라이언트

```python
client = StdioStatefulClient(
    name="filesystem",
    command="npx",
    args=["-y", "@modelcontextprotocol/server-filesystem", "/path"],
)

async with client:
    tools = await client.list_tools()
```

로컬 MCP 서버 프로세스를 직접 실행하여 통신.

### HTTP 클라이언트 (Stateful)

```python
client = HttpStatefulClient(
    name="remote-server",
    url="http://localhost:8080/sse",
)

async with client:
    tools = await client.list_tools()
```

SSE(Server-Sent Events) 기반 원격 MCP 서버 연동.

### HTTP 클라이언트 (Stateless)

```python
client = HttpStatelessClient(
    name="api-server",
    url="http://localhost:8080/mcp",
)
# 각 요청마다 새로운 HTTP 연결 사용
```

---

## Toolkit 연동

```python
from agentscope.mcp import StdioStatefulClient
from agentscope.tool import Toolkit

toolkit = Toolkit()
mcp_client = StdioStatefulClient(
    name="search",
    command="python",
    args=["./mcp_search_server.py"],
)

# MCP 서버의 모든 툴이 toolkit에 자동 등록
toolkit.register_mcp_client(mcp_client)

# 에이전트에서 사용
agent = ReActAgent(toolkit=toolkit, ...)
```

---

## MCP 콘텐츠 변환

MCP 서버 응답 → AgentScope 콘텐츠 블록:

| MCP 타입 | AgentScope 블록 |
|----------|----------------|
| `text` | `TextBlock` |
| `image` | `ImageBlock` |
| `audio` | `AudioBlock` |
| `resource` | `TextBlock` (URI 포함) |

---

## 모듈 연동

```
MCP
  ← Tool       (MCP 클라이언트 등록)
  → Message    (콘텐츠 블록 변환)
  ← Agent      (MCP 툴 호출)
```
