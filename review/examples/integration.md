# 통합 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/integration/`

서드파티 서비스 및 외부 모델과의 연동 예제.

---

## 1. qwen_deep_research_model — Qwen 딥리서치 모델

**파일**: `main.py`, `qwen_deep_research_agent.py`

**목적**: Qwen의 네이티브 딥리서치 API를 사용하는 에이전트. ReAct 루프 대신 모델 내장 딥리서치 기능 활용.

**실제 코드:**

```python
# examples/integration/qwen_deep_research_model/main.py
async def main():
    researcher = QwenDeepResearchAgent(
        name="Researcher Qwen",
        verbose=True,
    )

    # Step 1: 연구 방향 확인 질문
    user_msg = Msg(
        name="User",
        content="Research the applications of artificial intelligence in education",
        role="user",
    )
    clarification = await researcher(user_msg)
    print(f"\n{clarification.name}: {clarification.content}\n")

    # Step 2: 후속 질문에 응답 후 딥리서치 실행
    user_response = Msg(
        name="User",
        content="I am mainly interested in personalized learning and intelligent assessment.",
        role="user",
    )
    research_result = await researcher(user_response)
    print(f"\n{research_result.name}: {research_result.content}\n")
```

**`QwenDeepResearchAgent` 특징:**
- DashScope API의 `qwen-research` 또는 유사 딥리서치 엔드포인트 사용
- 2단계 워크플로우: (1) 사용자 의도 확인 → (2) 완전한 리서치 실행
- `verbose=True` — 내부 처리 과정 출력

---

## 2. alibabacloud_api_mcp — 알리바바 클라우드 MCP (OAuth 인증)

**파일**: `main.py`, `oauth_handler.py`

**목적**: Alibaba Cloud API MCP 서버에 OAuth 2.0 인증으로 연결하는 예제.

**실제 코드:**

```python
# examples/integration/alibabacloud_api_mcp/main.py
from mcp.client.auth import OAuthClientProvider
from mcp.shared.auth import OAuthClientMetadata
from oauth_handler import InMemoryTokenStorage, handle_callback, handle_redirect

# MCP 서버 엔드포인트 (api.aliyun.com에서 프로비저닝 후 취득)
server_url = (
    "https://openapi-mcp.cn-hangzhou.aliyuncs.com/accounts/14******/custom/"
    "****/id/KXy******/mcp"
)

memory_token_storage = InMemoryTokenStorage()

# OAuth 2.0 클라이언트 설정
oauth_provider = OAuthClientProvider(
    server_url=server_url,
    client_metadata=OAuthClientMetadata(
        client_name="AgentScopeExampleClient",
        redirect_uris=[AnyUrl("http://localhost:3000/callback")],
        grant_types=["authorization_code", "refresh_token"],
        response_types=["code"],
        scope=None,
    ),
    storage=memory_token_storage,
    redirect_handler=handle_redirect,
    callback_handler=handle_callback,
)

# OAuth 인증이 포함된 MCP 클라이언트
mcp_client = HttpStatelessClient(
    name="aliyun_api",
    transport="streamable_http",
    url=server_url,
    auth=oauth_provider,
)

toolkit = Toolkit()
await toolkit.register_mcp_client(mcp_client)

agent = ReActAgent(
    name="Friday",
    model=DashScopeChatModel(
        model_name="qwen-max",
        api_key=os.environ.get("DASHSCOPE_API_KEY"),
    ),
    formatter=DashScopeChatFormatter(),
    memory=InMemoryMemory(),
    toolkit=toolkit,
)

user = UserAgent("User")
msg = None
while True:
    msg = await user(msg)
    if msg.get_text_content() == "exit":
        break
    msg = await agent(msg)
```

**`oauth_handler.py`:**

```python
# examples/integration/alibabacloud_api_mcp/oauth_handler.py
class InMemoryTokenStorage:
    """메모리 내 토큰 저장소 (OAuth access/refresh 토큰)"""
    def __init__(self):
        self._tokens = {}

    async def get_tokens(self) -> dict | None:
        return self._tokens or None

    async def set_tokens(self, tokens: dict) -> None:
        self._tokens = tokens

async def handle_redirect(url: str) -> None:
    """OAuth 리다이렉트 URL을 브라우저에서 열기"""
    print(f"Please open the following URL in your browser:\n{url}")

async def handle_callback(url: str) -> str:
    """로컬 서버로 OAuth 콜백 수신"""
    # FastAPI/aiohttp 미니 서버로 authorization_code 수신
    ...
```

---

## 패턴 비교

| 통합 | 인증 방식 | 연결 방식 | 특징 |
|------|---------|---------|------|
| Qwen DeepResearch | API Key (DashScope) | 모델 직접 호출 | 2단계 리서치 워크플로우 |
| Alibaba Cloud MCP | OAuth 2.0 (Authorization Code Flow) | MCP StreamableHTTP | 토큰 자동 갱신 (refresh_token) |
