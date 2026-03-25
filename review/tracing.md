# Tracing 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

OpenTelemetry 기반의 분산 추적 시스템. 에이전트 실행, LLM 호출, 툴 실행, 임베딩 요청을 자동으로 추적하여 성능 모니터링과 디버깅을 지원한다. Arize-Phoenix, Langfuse 등 외부 관찰성 플랫폼과 연동된다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_trace.py` | 추적 데코레이터 정의 |
| `_setup.py` | OpenTelemetry 초기화 설정 |
| `_attributes.py` | 스팬 속성 상수 정의 |
| `_extractor.py` | 스팬 속성 추출 로직 |
| `_converter.py` | 응답 타입 변환 |
| `_utils.py` | 유틸리티 함수 |

---

## 핵심 데코레이터

```python
from agentscope.tracing import (
    trace_agent,
    trace_llm,
    trace_toolkit,
    trace_embedding,
)

@trace_agent
async def reply(self, *args, **kwargs) -> Msg:
    ...

@trace_llm
async def __call__(self, *args, **kwargs) -> ChatResponse:
    ...

@trace_toolkit
async def call_tool(self, tool_call: ToolUseBlock):
    ...

@trace_embedding
async def __call__(self, *args, **kwargs) -> EmbeddingResponse:
    ...
```

---

## 추적 범위

| 추적 대상 | 스팬 이름 | 기록 속성 |
|-----------|----------|----------|
| Agent.reply() | `agent.reply` | 에이전트명, 입력/출력 메시지 |
| ChatModel.__call__() | `llm.call` | 모델명, 프롬프트, 응답, 토큰 사용량 |
| Toolkit.call_tool() | `tool.call` | 툴명, 입력/출력 |
| EmbeddingModel.__call__() | `embedding.call` | 모델명, 입력 텍스트, 벡터 차원 |

---

## 설정

### OpenTelemetry 초기화

```python
from agentscope.tracing import setup_tracing

# OTLP exporter 설정
setup_tracing(
    endpoint="http://localhost:4317",    # OTLP gRPC 엔드포인트
    service_name="my-agentscope-app",
    trace_enabled=True,
)
```

### 컨텍스트 변수

```python
from agentscope.tracing import run_id, project, name, trace_enabled

# ContextVar 기반으로 스레드 안전하게 동작
run_id.set("session_123")
trace_enabled.set(True)
```

---

## 외부 플랫폼 연동

### Arize-Phoenix

```python
import phoenix as px

px.launch_app()  # 로컬 UI 실행

setup_tracing(
    endpoint="http://localhost:6006/v1/traces",
    service_name="agentscope",
)
```

### Langfuse

```python
from langfuse.opentelemetry import LangfuseExporter

setup_tracing(
    exporter=LangfuseExporter(
        public_key="...",
        secret_key="...",
        host="https://cloud.langfuse.com",
    ),
)
```

---

## 스팬 계층 구조

```
agent.reply (루트 스팬)
  ├─ llm.call (모델 호출)
  ├─ tool.call (툴 실행 1)
  │    └─ (툴 내부 로직)
  ├─ tool.call (툴 실행 2)
  └─ llm.call (최종 응답)
```

---

## 성능 메트릭

자동으로 수집되는 메트릭:
- **레이턴시**: 각 스팬의 실행 시간
- **토큰 사용량**: LLM 호출별 입력/출력 토큰
- **에러율**: 실패한 스팬의 상태 코드
- **툴 호출 빈도**: 툴별 호출 횟수 및 성공률

---

## 모듈 연동

```
Tracing
  ← Agent      (reply 추적)
  ← Model      (LLM 호출 추적)
  ← Tool       (툴 실행 추적)
  ← Embedding  (임베딩 추적)
  → OTLP       (외부 플랫폼으로 내보내기)
```
