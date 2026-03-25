# Embedding 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

텍스트와 멀티모달 콘텐츠를 벡터로 변환하는 임베딩 모델 모듈. OpenAI, Gemini, DashScope, Ollama 제공자를 지원하며, 파일 기반 캐싱으로 반복 요청 비용을 절감한다. RAG 및 장기 메모리의 의미 검색에 활용된다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_embedding_base.py` | 추상 임베딩 모델 기반 클래스 |
| `_embedding_response.py` | 임베딩 응답 구조 |
| `_embedding_usage.py` | 토큰 사용량 추적 |
| `_openai_embedding.py` | OpenAI 임베딩 |
| `_dashscope_embedding.py` | DashScope 텍스트 임베딩 |
| `_dashscope_multimodal_embedding.py` | DashScope 멀티모달 임베딩 |
| `_gemini_embedding.py` | Google Gemini 임베딩 |
| `_ollama_embedding.py` | 로컬 Ollama 임베딩 |
| `_cache_base.py` | 캐시 추상 기반 클래스 |
| `_file_cache.py` | 파일 기반 캐시 구현 |

---

## 핵심 클래스

### `EmbeddingModelBase`

```python
class EmbeddingModelBase:
    model_name: str
    dimensions: int | None              # 벡터 차원 수
    supported_modalities: list[str]     # ["text", "image", "video"]

    async def __call__(
        *args, **kwargs
    ) -> EmbeddingResponse
```

---

### `EmbeddingResponse`

```python
class EmbeddingResponse:
    embeddings: list[list[float]]   # 임베딩 벡터 리스트
    usage: EmbeddingUsage
    metadata: dict
```

---

### `EmbeddingUsage`

```python
class EmbeddingUsage:
    input_tokens: int
    time: float
```

---

## 지원 제공자

| 제공자 | 모델 예시 | 차원 | 멀티모달 |
|--------|----------|------|---------|
| OpenAI | `text-embedding-3-small` | 1536 | 텍스트만 |
| OpenAI | `text-embedding-3-large` | 3072 | 텍스트만 |
| DashScope | `text-embedding-v3` | 1024 | 텍스트만 |
| DashScope | `multimodal-embedding-one-peace-v1` | 1536 | 텍스트+이미지+오디오 |
| Gemini | `text-embedding-004` | 768 | 텍스트만 |
| Ollama | `nomic-embed-text` | 768 | 텍스트만 |

---

## 캐싱 시스템

동일한 입력에 대한 임베딩을 파일에 캐싱하여 API 호출 비용 절감:

```python
from agentscope.embedding import OpenAIEmbeddingModel
from agentscope.embedding import FileCache

cache = FileCache(cache_dir="./embedding_cache/")

embedding_model = OpenAIEmbeddingModel(
    model_name="text-embedding-3-small",
    cache=cache,
)

# 첫 번째 호출: API 요청 후 캐싱
response = await embedding_model("안녕하세요")

# 두 번째 호출: 캐시에서 즉시 반환 (API 요청 없음)
response = await embedding_model("안녕하세요")
```

캐시 키: 입력 텍스트의 해시값 + 모델명

---

## 멀티모달 임베딩

```python
from agentscope.embedding import DashScopeMultimodalEmbeddingModel
from agentscope.message import ImageBlock, URLSource

model = DashScopeMultimodalEmbeddingModel(
    model_name="multimodal-embedding-one-peace-v1",
)

# 이미지 임베딩
response = await model(
    ImageBlock(
        type="image",
        source=URLSource(type="url", url="https://example.com/image.jpg")
    )
)

# 텍스트-이미지 결합 임베딩
response = await model(
    ["고양이 사진", ImageBlock(...)]
)
```

---

## RAG 연동

```python
from agentscope.rag import KnowledgeBase
from agentscope.embedding import OpenAIEmbeddingModel

embedding_model = OpenAIEmbeddingModel(
    model_name="text-embedding-3-small",
)

knowledge_base = KnowledgeBase(
    embedding_model=embedding_model,
    embedding_store=qdrant_store,
)

# 문서 임베딩 후 저장
await knowledge_base.add_documents(documents)

# 쿼리 임베딩 후 유사 문서 검색
results = await knowledge_base.retrieve("검색어", limit=5)
```

---

## 모듈 연동

```
Embedding
  ← RAG        (문서/쿼리 벡터화)
  ← Memory     (장기 메모리 의미 검색)
  → Tracing    (임베딩 호출 추적)
  → Cache      (반복 요청 최적화)
```
