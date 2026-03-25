# Hooks 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

에이전트 실행 흐름에 외부 기능을 삽입할 수 있는 이벤트 훅 시스템. AgentScope Studio와의 실시간 연동 훅이 기본 제공되며, 커스텀 훅을 통해 로깅, 모니터링, 메시지 변환 등 다양한 기능을 에이전트에 주입할 수 있다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_studio_hooks.py` | AgentScope Studio 연동 훅 |
| `__init__.py` | 훅 등록 및 내보내기 |

---

## 훅 시스템 아키텍처

### 메타클래스 기반 자동 등록

`_AgentMeta` 메타클래스가 에이전트 클래스 정의 시 훅을 자동으로 래핑:

```python
class _AgentMeta(type):
    def __new__(mcs, name, bases, namespace):
        # reply(), observe(), print() 등을 훅으로 래핑
        # pre_* 훅 → 원본 메서드 → post_* 훅
        ...
```

---

## 훅 타입

### 에이전트 핵심 훅

| 훅 이름 | 실행 시점 |
|---------|----------|
| `pre_reply` | `reply()` 호출 전 |
| `post_reply` | `reply()` 반환 후 |
| `pre_observe` | `observe()` 호출 전 |
| `post_observe` | `observe()` 반환 후 |
| `pre_print` | `print()` 호출 전 |
| `post_print` | `print()` 반환 후 |

### ReAct 에이전트 추가 훅

| 훅 이름 | 실행 시점 |
|---------|----------|
| `pre_reasoning` | 추론(Think) 단계 전 |
| `post_reasoning` | 추론 단계 후 |
| `pre_acting` | 행동(Act) 단계 전 |
| `post_acting` | 행동 단계 후 |

---

## Studio 훅

### `as_studio_forward_message_pre_print_hook`

```python
def as_studio_forward_message_pre_print_hook(studio_url: str) -> Callable:
    """에이전트가 메시지를 출력하기 전 Studio로 전달하는 훅 반환"""

    async def hook(agent, msg, last, speech):
        # AgentScope Studio WebSocket으로 메시지 전송
        await studio_client.send(msg)
        return msg, last, speech

    return hook
```

### Studio 전체 연동

```python
from agentscope.hooks import equip_as_studio_hooks

# 에이전트의 모든 출력을 Studio로 실시간 전달
equip_as_studio_hooks(
    agent=my_agent,
    studio_url="http://localhost:7862",
)
```

---

## 커스텀 훅 등록

```python
# pre_reply 훅: 입력 메시지 전처리
async def my_pre_reply_hook(agent, *args, **kwargs):
    # 입력 로깅
    logger.info(f"[{agent.name}] 입력: {args}")
    return args, kwargs

# post_reply 훅: 출력 메시지 후처리
async def my_post_reply_hook(agent, response: Msg) -> Msg:
    # 응답에 타임스탬프 추가
    response.metadata["processed_at"] = datetime.now().isoformat()
    return response

# 훅 등록
my_agent.pre_reply.append(my_pre_reply_hook)
my_agent.post_reply.append(my_post_reply_hook)
```

---

## 훅 실행 순서

```
reply() 호출
  → pre_reply hooks[0]
  → pre_reply hooks[1]
  → ...
    → 에이전트 내부 로직 실행
  → post_reply hooks[0]
  → post_reply hooks[1]
  → ...
  → 최종 Msg 반환
```

여러 훅이 등록된 경우 리스트 순서대로 순차 실행.

---

## 활용 사례

| 활용 | 훅 타입 |
|------|---------|
| 입력 검열/필터링 | `pre_reply` |
| 응답 포맷 변환 | `post_reply` |
| 메모리 강제 추가 | `post_observe` |
| 실시간 UI 업데이트 | `pre_print` |
| 감사 로깅 | `pre_reply`, `post_reply` |
| 레이트 리밋 | `pre_reply` |

---

## 모듈 연동

```
Hooks
  ← Agent      (훅 등록 대상)
  → Studio     (메시지 실시간 전달)
  → Tracing    (훅 실행 추적 가능)
```
