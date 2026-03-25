# A2A 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

Agent-to-Agent(A2A) 프로토콜을 구현하는 모듈. 원격 에이전트를 발견(discovery)하고, 에이전트 간 메시지를 교환하는 표준 인터페이스를 제공한다. 파일, Nacos, Well-known 엔드포인트 등 다양한 디스커버리 방식을 지원한다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_base.py` | 에이전트 카드 리졸버 추상 인터페이스 |
| `_file_resolver.py` | 파일 기반 에이전트 디스커버리 |
| `_nacos_resolver.py` | Nacos 서비스 레지스트리 연동 |
| `_well_known_resolver.py` | Well-known URL 엔드포인트 |

---

## 핵심 클래스

### `AgentCardResolverBase`

```python
class AgentCardResolverBase:
    async def get_agent_card(
        *args, **kwargs
    ) -> AgentCard
```

원격 에이전트의 메타데이터(AgentCard)를 조회하는 추상 인터페이스.

---

### `AgentCard`

원격 에이전트의 명세서:
- 에이전트 이름 및 설명
- 지원하는 입력/출력 타입
- 엔드포인트 URL
- 제공하는 스킬/능력 목록

---

## 디스커버리 방식

### 파일 기반

```python
resolver = FileAgentCardResolver(
    card_path="./agents/search_agent.json"
)
agent_card = await resolver.get_agent_card()
```

로컬 JSON 파일에서 에이전트 카드를 로드. 개발/테스트 환경에 적합.

### Nacos 레지스트리

```python
resolver = NacosAgentCardResolver(
    server_addr="http://nacos:8848",
    service_name="search-agent",
    namespace="prod",
)
agent_card = await resolver.get_agent_card()
```

Nacos 서비스 디스커버리를 통해 동적으로 에이전트 위치를 파악. 마이크로서비스 환경에 적합.

### Well-known 엔드포인트

```python
resolver = WellKnownAgentCardResolver(
    base_url="http://search-agent:8080"
)
# GET http://search-agent:8080/.well-known/agent-card 요청
agent_card = await resolver.get_agent_card()
```

HTTP 표준 Well-known 경로에서 에이전트 카드를 조회.

---

## A2AAgent 연동

```python
from agentscope.agent import A2AAgent
from agentscope.a2a import FileAgentCardResolver

agent = A2AAgent(
    name="orchestrator",
    resolver=FileAgentCardResolver("./remote_agent.json"),
)

# 원격 에이전트에 메시지 전송
response = await agent.reply(msg)
```

내부적으로 A2A Formatter를 사용하여 메시지를 A2A 프로토콜 포맷으로 변환.

---

## 메시지 흐름

```
A2AAgent.reply(Msg)
  ↓
A2AFormatter.format()        # Msg → A2A 프로토콜 포맷
  ↓
AgentCardResolver.get_agent_card()  # 원격 에이전트 주소 조회
  ↓
HTTP 요청 (원격 에이전트 엔드포인트)
  ↓
응답 → Msg 변환
  ↓
return Msg
```

---

## 모듈 연동

```
A2A
  ← Agent      (A2AAgent가 리졸버 사용)
  → Formatter  (A2A 포맷 변환)
  → Message    (Msg 직렬화/역직렬화)
```
