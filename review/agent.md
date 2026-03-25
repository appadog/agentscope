# Agent 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

에이전트의 핵심 구현체를 담은 모듈. `AgentBase`를 중심으로 ReAct, A2A, 실시간, 사용자 에이전트 등 다양한 에이전트 유형을 제공한다. Hook 기반 확장 시스템과 비동기 실행을 핵심 설계 원칙으로 한다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_agent_base.py` | 모든 에이전트의 추상 기반 클래스, Hook 시스템 포함 |
| `_agent_meta.py` | Hook 자동 등록을 위한 메타클래스 |
| `_react_agent_base.py` | ReAct 패턴 기반 클래스 (reasoning/acting 훅 추가) |
| `_react_agent.py` | 완전한 ReAct 구현체 (메모리 압축, 쿼리 재작성 포함) |
| `_user_agent.py` | 사용자 입력을 처리하는 에이전트 |
| `_user_input.py` | 사용자 입력 추상화 레이어 |
| `_a2a_agent.py` | Agent-to-Agent 프로토콜 에이전트 |
| `_realtime_agent.py` | WebSocket 기반 실시간 음성 에이전트 |
| `_utils.py` | 비동기 컨텍스트 유틸리티 |

---

## 핵심 클래스

### `AgentBase`

```python
class AgentBase(StateModule, metaclass=_AgentMeta):
    name: str
    model: ChatModelBase
    memory: MemoryBase | None
    toolkit: Toolkit | None
    knowledge_base: KnowledgeBase | None
    sys_prompt: str | None

    async def reply(*args, **kwargs) -> Msg
    async def observe(msg: Msg | list[Msg] | None) -> None
    async def print(msg: Msg, last: bool, speech: AudioBlock | None) -> None
```

모든 에이전트의 추상 기반 클래스. `StateModule`을 상속하여 직렬화/복원이 가능하고, `_AgentMeta` 메타클래스를 통해 Hook이 자동 등록된다.

**Hook 라이프사이클**:
```
reply() 호출
  → pre_reply hooks
    → (실제 추론 실행)
  → post_reply hooks
  → return Msg

observe() 호출
  → pre_observe hooks
    → (메모리에 메시지 저장)
  → post_observe hooks

print() 호출
  → pre_print hooks
    → (출력 렌더링)
  → post_print hooks
```

---

### `ReActAgentBase`

```python
class ReActAgentBase(AgentBase, metaclass=_ReActAgentMeta):
    # 추가 훅
    pre_reasoning / post_reasoning
    pre_acting / post_acting
```

Think → Act → Observe 루프를 구현한 추상 클래스. 병렬 툴 호출(parallel tool calling) 지원.

---

### `ReActAgent`

`ReActAgentBase`의 완전한 구현체.

**주요 기능**:
- **메모리 압축**: 대화가 길어지면 요약을 생성하고 오래된 메시지를 압축
- **쿼리 재작성**: 지식 검색 시 질의를 최적화하여 RAG 정확도 향상
- **마크 시스템**: 메모리 메시지에 마크를 붙여 선택적 검색 지원

```python
class ReActAgent(ReActAgentBase):
    compress_config: CompressConfig | None  # 메모리 압축 설정
    summary_schema: dict | None             # 압축 요약 스키마
```

---

### `UserAgent`

```python
class UserAgent(AgentBase):
    input_func: Callable  # 입력 방식 (stdin, GUI, etc.)
```

사용자로부터 입력을 받아 Msg로 변환. 입력 함수를 교체하여 다양한 인터페이스를 지원.

---

### `A2AAgent`

```python
class A2AAgent(AgentBase):
    resolver: AgentCardResolverBase  # 원격 에이전트 디스커버리
```

원격 에이전트와 A2A 프로토콜로 통신. 내부적으로 A2A 포맷 변환기를 사용.

---

### `RealtimeAgent`

```python
class RealtimeAgent(AgentBase):
    realtime_model: RealtimeModelBase
    tts_model: TTSModelBase | None
```

WebSocket 기반 실시간 양방향 대화. 오디오 스트림 입출력을 처리하며 `ChatRoom` 파이프라인과 연동.

---

## 모듈 연동

```
AgentBase
  ├─ Model      → LLM API 호출
  ├─ Memory     → 대화 이력 관리
  ├─ Tool       → 함수 실행
  ├─ Formatter  → 메시지 → API 포맷 변환
  ├─ RAG        → 지식 검색
  ├─ Tracing    → 실행 추적
  └─ Session    → 상태 영속성
```

---

## 설계 패턴

- **Template Method**: `reply()`는 추상 메서드, 서브클래스가 구현
- **Metaclass Hook 등록**: `_AgentMeta`가 훅을 자동으로 래핑
- **StateModule**: 에이전트 전체 상태를 직렬화/역직렬화 가능
