# Examples 개요

> [← 요약으로 돌아가기](../summary.md)

## 구조

```
examples/
├── agent/            # 에이전트 타입별 구현 예제
├── functionality/    # 개별 기능 시연 예제
├── workflows/        # 멀티 에이전트 워크플로우
├── deployment/       # 프로덕션 배포 패턴
├── game/             # 게임 기반 시뮬레이션
├── evaluation/       # 평가 및 벤치마킹
├── tuner/            # 최적화 및 파인튜닝
└── integration/      # 서드파티 서비스 연동
```

---

## 카테고리별 요약

| 카테고리 | 예제 수 | 핵심 내용 | 상세 |
|----------|---------|-----------|------|
| [에이전트](./agents.md) | 8 | ReAct, A2A, 음성, 브라우저, 딥리서치 | [→](./agents.md) |
| [기능](./functionality.md) | 13 | RAG, 메모리, MCP, TTS, 플래닝 | [→](./functionality.md) |
| [워크플로우](./workflows.md) | 4 | 멀티에이전트 대화/토론/동시실행 | [→](./workflows.md) |
| [배포](./deployment.md) | 1 | Quart 서버, SSE 스트리밍 | [→](./deployment.md) |
| [게임](./game.md) | 1 | 늑대인간 게임 | [→](./game.md) |
| [평가](./evaluation.md) | 1 | ACE-Bench, Ray 분산 평가 | [→](./evaluation.md) |
| [튜너](./tuner.md) | 3 | 프롬프트 튜닝, 모델 선택 | [→](./tuner.md) |
| [통합](./integration.md) | 2 | Qwen, 알리바바 클라우드 | [→](./integration.md) |

---

## 공통 패턴

모든 예제에서 반복적으로 사용되는 핵심 패턴들.

### 에이전트 생성

```python
agent = ReActAgent(
    name="assistant",
    sys_prompt="당신은 도움이 되는 AI 어시스턴트입니다.",
    model=DashScopeChatModel(model_name="qwen-max"),
    formatter=DashScopeChatFormatter(),
    toolkit=Toolkit(),
    memory=InMemoryMemory(),
)
```

### 메시지 전송 및 응답 수신

```python
msg = Msg(name="user", content="안녕하세요", role="user")
response = await agent(msg)
print(response.content)
```

### 툴 등록

```python
toolkit = Toolkit()
toolkit.register_tool_function(my_function)
toolkit.register_tool_function(execute_python_code)
await toolkit.register_mcp_client(mcp_client)
```

### 스트리밍 출력

```python
async for msg, last in stream_printing_messages(
    agents=[agent],
    coroutine_task=agent(user_msg),
):
    # 실시간으로 메시지 출력
    pass
```

### 세션 저장/복원

```python
session = JSONSession(save_dir="./sessions/")
await session.load_session_state(session_id="s1", agent=agent)
# ... 대화 ...
await session.save_session_state(session_id="s1", agent=agent)
```

### 멀티 에이전트 허브

```python
async with MsgHub(participants=[agent1, agent2, agent3]) as hub:
    await sequential_pipeline([agent1, agent2, agent3], initial_msg)
```
