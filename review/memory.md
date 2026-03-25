# Memory 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

에이전트의 대화 이력(단기 메모리)과 지식 저장소(장기 메모리)를 관리하는 모듈. 다양한 백엔드(인메모리, SQLite, Redis, mem0ai, ReMe)를 지원하며, 마크(mark) 기반 선택적 검색을 핵심 기능으로 제공한다.

---

## 파일 구성

### 워킹 메모리 (단기)

| 파일 | 역할 |
|------|------|
| `_working_memory/_base.py` | 추상 기반 클래스 |
| `_working_memory/_in_memory_memory.py` | 인메모리 구현 |
| `_working_memory/_sqlalchemy_memory.py` | SQL DB 백엔드 |
| `_working_memory/_redis_memory.py` | Redis 백엔드 |

### 장기 메모리

| 파일 | 역할 |
|------|------|
| `_long_term_memory/_long_term_memory_base.py` | 추상 기반 클래스 |
| `_long_term_memory/_mem0/` | mem0ai 통합 |
| `_long_term_memory/_reme/` | ReMe (개인/작업/툴 메모리) |

---

## 워킹 메모리 (단기 메모리)

### `MemoryBase`

```python
class MemoryBase(StateModule):
    _compressed_summary: str   # 압축 캐시

    async def add(
        memories: list[Msg],
        marks: list[str] | None
    ) -> None

    async def delete(msg_ids: list[str]) -> None
    async def size() -> int

    async def get_memory(
        mark: str | None = None,
        exclude_mark: str | None = None,
        prepend_summary: bool = True
    ) -> list[Msg]

    async def update_messages_mark(
        new_mark: str,
        old_mark: str | None,
        msg_ids: list[str] | None
    ) -> None
```

**마크(Mark) 시스템**:
메시지에 태그를 붙여 선택적으로 검색 가능. ReAct 에이전트에서 추론/행동 단계를 구분하거나, 특정 대화 세션을 분리하는 데 활용.

```python
# 마크 붙여서 저장
await memory.add([msg], marks=["task_1"])

# 특정 마크의 메시지만 조회
msgs = await memory.get_memory(mark="task_1")

# 특정 마크 제외하고 조회
msgs = await memory.get_memory(exclude_mark="temporary")
```

---

### 지원 백엔드

| 백엔드 | 특징 | 사용 시나리오 |
|--------|------|--------------|
| InMemory | 빠름, 프로세스 종료 시 소멸 | 개발/테스트 |
| SQLAlchemy | 영속적, SQL 쿼리 지원 | 프로덕션 단일 인스턴스 |
| Redis | 빠름, 분산 환경 지원 | 프로덕션 멀티 인스턴스 |

---

## 장기 메모리

### `LongTermMemoryBase`

```python
class LongTermMemoryBase(StateModule):
    async def record(msgs: list[Msg], **kwargs) -> None
    async def retrieve(msg: Msg, limit: int) -> list[Msg]

    # 에이전트 툴로 노출되는 함수들
    async def record_to_memory(
        thinking: str,
        content: str
    ) -> str

    async def retrieve_from_memory(
        keywords: list[str],
        limit: int = 5
    ) -> str
```

---

### mem0ai 통합

```python
class Mem0LongTermMemory(LongTermMemoryBase):
    # mem0ai의 벡터 기반 의미 검색 사용
    # 자동 중복 제거 및 메모리 업데이트
```

---

### ReMe (Retrieval-Enhanced Memory)

```python
# 3가지 메모리 유형으로 구조화
class PersonalMemory   # 사용자 개인 정보
class TaskMemory       # 작업 관련 정보
class ToolMemory       # 툴 사용 패턴
```

ReMe는 단순 키워드 검색을 넘어 메모리의 구조화된 저장과 의미 기반 검색을 제공. 장기 대화나 지속적인 작업에서 특히 효과적.

---

## 메모리 압축

`ReActAgent`의 `CompressConfig`와 연동하여 대화가 일정 길이를 초과하면 자동 압축:

```python
class CompressConfig:
    max_tokens: int           # 압축 트리거 임계값
    compress_ratio: float     # 압축 후 목표 비율
    summary_schema: dict      # 요약 구조 스키마

# 압축 과정:
# 1. 오래된 메시지들을 LLM으로 요약
# 2. 요약을 _compressed_summary에 저장
# 3. 원본 메시지 삭제
# 4. get_memory() 호출 시 요약을 앞에 추가
```

---

## 모듈 연동

```
MemoryBase
  ← Agent      (메시지 저장/조회)
  ← ReActAgent (메모리 압축 트리거)
  → Session    (세션 영속성)
  → Tracing    (메모리 작업 추적)

LongTermMemoryBase
  ← Agent      (툴로 노출)
  → Embedding  (의미 검색용 임베딩)
  → RAG        (지식 검색 연동)
```
