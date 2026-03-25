# AgentScope 코드베이스 분석

> **버전**: 1.0.17 | **라이선스**: Apache 2.0 | **Python**: 3.10+

AgentScope는 LLM 기반 멀티 에이전트 애플리케이션을 위한 프로덕션 레디 파이썬 프레임워크입니다.
에이전트 지향 프로그래밍(AOP) 패러다임을 기반으로 하며, 복잡한 멀티 에이전트 워크플로우를 5분 안에 구축할 수 있도록 설계되어 있습니다.

---

## 전체 구조

```
src/agentscope/
├── agent/        # 에이전트 구현체
├── model/        # LLM 모델 통합
├── message/      # 메시지 및 콘텐츠 블록
├── formatter/    # 모델별 메시지 포맷 변환
├── tool/         # 툴 프레임워크
├── memory/       # 단기/장기 메모리
├── rag/          # RAG (검색 증강 생성)
├── mcp/          # Model Context Protocol
├── a2a/          # 에이전트 간 통신
├── realtime/     # 실시간 음성 에이전트
├── tts/          # 텍스트-음성 변환
├── tracing/      # 분산 추적 (OpenTelemetry)
├── pipeline/     # 메시지 오케스트레이션
├── session/      # 세션 영속성
├── evaluate/     # 평가 프레임워크
├── tuner/        # 모델/프롬프트 튜닝
├── hooks/        # 이벤트 훅 시스템
└── embedding/    # 임베딩 모델
```

**[→ 단계별 사용 가이드](./guide.md)**

---

## 핵심 모듈 분석

| 모듈 | 역할 | 상세 분석 |
|------|------|-----------|
| [agent](./agent.md) | ReAct, A2A, 실시간, 사용자 에이전트 구현 | [→](./agent.md) |
| [model](./model.md) | OpenAI, Anthropic, Gemini, DashScope, Ollama 통합 | [→](./model.md) |
| [message](./message.md) | 멀티모달 메시지 구조 (텍스트/이미지/오디오/비디오/툴) | [→](./message.md) |
| [formatter](./formatter.md) | 모델별 API 포맷 변환 레이어 | [→](./formatter.md) |
| [tool](./tool.md) | 툴 등록, 실행, MCP/스킬 관리 | [→](./tool.md) |
| [memory](./memory.md) | 워킹 메모리 + Redis/SQL/ReMe 장기 메모리 | [→](./memory.md) |
| [rag](./rag.md) | 문서 리더, 벡터 DB, 지식 베이스 | [→](./rag.md) |
| [mcp](./mcp.md) | MCP 클라이언트 (stdio/HTTP) | [→](./mcp.md) |
| [a2a](./a2a.md) | 에이전트 간 프로토콜 및 디스커버리 | [→](./a2a.md) |
| [realtime](./realtime.md) | WebSocket 기반 실시간 음성 에이전트 | [→](./realtime.md) |
| [tts](./tts.md) | 텍스트-음성 합성 (스트리밍 지원) | [→](./tts.md) |
| [tracing](./tracing.md) | OpenTelemetry 기반 분산 추적 | [→](./tracing.md) |
| [pipeline](./pipeline.md) | MsgHub, ChatRoom 오케스트레이션 | [→](./pipeline.md) |
| [session](./session.md) | JSON/Redis 기반 세션 영속성 | [→](./session.md) |
| [evaluate](./evaluate.md) | 벤치마크, 메트릭, 분산 평가 | [→](./evaluate.md) |
| [tuner](./tuner.md) | 프롬프트/모델 자동 최적화 | [→](./tuner.md) |
| [hooks](./hooks.md) | Studio 연동 pre/post 훅 | [→](./hooks.md) |
| [embedding](./embedding.md) | 임베딩 모델 + 캐싱 | [→](./embedding.md) |

---

## 예제 분석 (examples/)

> 8개 카테고리, 92개 파이썬 파일 — [전체 예제 개요](./examples/overview.md)

| 카테고리 | 내용 | 상세 분석 |
|----------|------|-----------|
| [에이전트 예제](./examples/agents.md) | ReAct, A2A, 실시간 음성, 브라우저, 딥리서치, 메타 플래너 | [→](./examples/agents.md) |
| [기능 예제](./examples/functionality.md) | RAG, 메모리, 구조화 출력, 플래닝, MCP, TTS, 세션 | [→](./examples/functionality.md) |
| [워크플로우 예제](./examples/workflows.md) | 멀티에이전트 대화, 토론, 동시 실행, 실시간 | [→](./examples/workflows.md) |
| [배포 예제](./examples/deployment.md) | Quart 기반 프로덕션 서버, SSE 스트리밍 | [→](./examples/deployment.md) |
| [게임 예제](./examples/game.md) | 늑대인간 게임 시뮬레이션 | [→](./examples/game.md) |
| [평가 예제](./examples/evaluation.md) | ACE-Bench, Ray 분산 평가 | [→](./examples/evaluation.md) |
| [튜너 예제](./examples/tuner.md) | 프롬프트 튜닝, 모델 선택, 파인튜닝 | [→](./examples/tuner.md) |
| [통합 예제](./examples/integration.md) | Qwen 딥리서치, 알리바바 클라우드 MCP | [→](./examples/integration.md) |

---

## 테스트 분석 (tests/)

> 54개 테스트 파일, 모든 핵심 모듈 커버 — [전체 테스트 개요](./tests/overview.md)

| 카테고리 | 파일 수 | 상세 분석 |
|----------|---------|-----------|
| [에이전트 테스트](./tests/agents.md) | 5 | AgentBase, ReAct, A2A 훅/통신 | [→](./tests/agents.md) |
| [포매터 테스트](./tests/formatters.md) | 7 | OpenAI, Anthropic, Gemini, DashScope 포맷 변환 | [→](./tests/formatters.md) |
| [모델 테스트](./tests/models.md) | 5 | LLM API 통합, 스트리밍, 구조화 출력 | [→](./tests/models.md) |
| [메모리 테스트](./tests/memory.md) | 4 | 워킹 메모리, 압축, ReMe, mem0 | [→](./tests/memory.md) |
| [툴 테스트](./tests/tools.md) | 5 | 코드 실행, 파일 조작, Toolkit | [→](./tests/tools.md) |
| [RAG 테스트](./tests/rag.md) | 3 | 문서 리더, 벡터 스토어, 지식 베이스 | [→](./tests/rag.md) |
| [실시간 테스트](./tests/realtime.md) | 4 | WebSocket, 이벤트 변환, 음성 | [→](./tests/realtime.md) |
| [추적 테스트](./tests/tracing.md) | 4 | 데코레이터, 스팬 변환, 추출 | [→](./tests/tracing.md) |
| [기타 테스트](./tests/others.md) | 17 | TTS, MCP, 토큰, 세션, 파이프라인 등 | [→](./tests/others.md) |

---

## 아키텍처 개요

```
┌─────────────────────────────────────────────────┐
│                   Agent Layer                    │
│  ReActAgent / UserAgent / A2AAgent / Realtime   │
└────────┬──────────┬──────────┬──────────────────┘
         │          │          │
    ┌────▼───┐  ┌───▼────┐  ┌─▼──────────┐
    │ Memory │  │  Tool  │  │  Pipeline  │
    │  RAG   │  │  MCP   │  │  MsgHub    │
    └────────┘  └───┬────┘  └────────────┘
                    │
         ┌──────────▼──────────┐
         │   Formatter Layer   │
         │ (OpenAI/Anthropic/  │
         │  Gemini/DashScope)  │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │    Model Layer      │
         │  ChatModelBase      │
         │  RealtimeModelBase  │
         └─────────────────────┘
                    │
         ┌──────────▼──────────┐
         │  Infrastructure     │
         │  Tracing / Session  │
         │  Evaluate / Tuner   │
         └─────────────────────┘
```

---

## 핵심 설계 패턴

1. **Hook 시스템** — `pre_reply` / `post_reply` / `pre_observe` 등 에이전트 라이프사이클 확장
2. **Formatter 패턴** — 모델 제공자별 메시지 변환을 독립 레이어로 분리
3. **StateModule** — 모든 핵심 컴포넌트의 직렬화/복원 지원
4. **Middleware Chain** — 툴 실행 파이프라인에 composable 미들웨어 적용
5. **비동기 우선** — `asyncio` 기반 동시 에이전트 실행
6. **Mark 기반 메모리** — 메모리에 마크를 붙여 선택적 검색 지원

---

## 주요 의존성

### 코어
- `openai`, `anthropic`, `dashscope`, `google-genai` — 모델 API
- `mcp>=1.13` — Model Context Protocol
- `opentelemetry-*` — 분산 추적

### 선택적
| 그룹 | 주요 패키지 |
|------|------------|
| memory | `redis`, `mem0ai`, `reme-ai` |
| rag | `qdrant-client`, `pymilvus`, `pymongo` |
| realtime | `websockets>=14.0`, `scipy` |
| tuner | `dspy>=3.1.0`, `datasets>=4.0.0` |
| evaluate | `ray` |

---

## 최근 주요 업데이트

| 날짜 | 기능 |
|------|------|
| 2026.02 | 실시간 음성 에이전트 |
| 2025.12 | A2A 프로토콜, TTS 지원 |
| 2025.11 | Anthropic Agent Skills |
| 2025.xx | Agentic RL (Trinity-RFT), ReMe 장기 메모리 |
