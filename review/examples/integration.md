# 통합 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/integration/`

서드파티 서비스 및 외부 모델과의 연동 예제.

---

## 1. qwen_deep_research_model — Qwen 딥리서치 모델

**파일**: `main.py`, `qwen_deep_research_agent.py`

**목적**: Qwen의 네이티브 딥리서치 모델을 사용하는 에이전트. 단순 ReAct 루프 대신 모델 자체의 딥리서치 기능을 활용.

```python
class QwenDeepResearchAgent(AgentBase):
    """Qwen 딥리서치 모델 전용 에이전트"""

    async def reply(self, msg: Msg) -> Msg:
        # 1단계: 리서치 주제 명확화 (클라리피케이션)
        clarification_response = await self.model(
            messages=[...],
            mode="clarification",
        )

        if clarification_response.needs_clarification:
            # 사용자에게 추가 정보 요청
            clarification_msg = Msg(
                name=self.name,
                content=clarification_response.questions,
                role="assistant",
            )
            user_followup = await self.user_agent.reply(clarification_msg)

        # 2단계: 딥리서치 실행
        research_response = await self.model(
            messages=[...],
            mode="deep_research",
        )

        return Msg(
            name=self.name,
            content=research_response.report,
            role="assistant",
        )

# 사용 예시
agent = QwenDeepResearchAgent(
    name="researcher",
    model=DashScopeChatModel(
        model_name="qwq-plus",  # 딥리서치 지원 모델
    ),
)

result = await agent.reply(
    Msg(name="user", content="양자 컴퓨팅의 현재 발전 상황을 조사해줘", role="user")
)
```

**멀티턴 구조**:
```
사용자 질문
  ↓
명확화 단계 (모델이 추가 정보 요청)
  ↓
사용자 답변
  ↓
딥리서치 실행 (모델이 내부적으로 검색/분석)
  ↓
리서치 보고서 반환
```

---

## 2. alibabacloud_api_mcp — 알리바바 클라우드 API MCP

**파일**: `main.py`, `oauth_handler.py`

**목적**: OAuth 인증을 통해 알리바바 클라우드 서비스 API에 접근하는 MCP 통합 예제.

```python
from agentscope.mcp import StdIOStatefulClient

# OAuth 핸들러 초기화
oauth = AliyunOAuthHandler(
    client_id="YOUR_CLIENT_ID",
    client_secret="YOUR_CLIENT_SECRET",
    region="cn-hangzhou",
)

# OAuth 토큰 획득
token = await oauth.get_access_token()

# 알리바바 클라우드 MCP 서버 연결
aliyun_client = StdIOStatefulClient(
    name="aliyun_api",
    command="npx",
    args=["@alibabacloud/mcp-server"],
    env={
        "ALIBABA_CLOUD_ACCESS_TOKEN": token,
        "REGION": "cn-hangzhou",
    },
)

await aliyun_client.connect()
toolkit = Toolkit()
await toolkit.register_mcp_client(aliyun_client)

# 클라우드 서비스를 툴로 사용하는 에이전트
agent = ReActAgent(
    name="cloud_agent",
    toolkit=toolkit,
    ...
)

# 에이전트가 ECS, OSS, RDS 등 클라우드 서비스 API를 직접 호출
response = await agent.reply(
    Msg(name="user", content="현재 ECS 인스턴스 목록을 조회해줘", role="user")
)
```

**OAuth 흐름**:
```
클라이언트 ID/시크릿
  ↓
AliyunOAuthHandler.get_access_token()
  ↓
알리바바 클라우드 OAuth 서버
  ↓
Access Token
  ↓
MCP 서버 환경변수로 주입
  ↓
MCP 서버가 토큰으로 API 인증
```

---

## 통합 패턴 요약

| 예제 | 통합 대상 | 방식 | 핵심 포인트 |
|------|----------|------|------------|
| Qwen Deep Research | Qwen 딥리서치 API | 커스텀 에이전트 클래스 | 멀티턴 명확화 → 리서치 |
| Aliyun API MCP | 알리바바 클라우드 서비스 | MCP + OAuth | OAuth 인증 후 MCP 연동 |
