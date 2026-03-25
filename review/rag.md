# RAG 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

검색 증강 생성(RAG)을 위한 문서 관리, 임베딩, 벡터 데이터베이스 통합 모듈. PDF, DOCX, Excel, PPT 등 다양한 문서 형식을 읽어 벡터 DB에 저장하고, 에이전트가 질의할 수 있는 지식 베이스를 구성한다.

---

## 파일 구성

### 코어

| 파일 | 역할 |
|------|------|
| `_knowledge_base.py` | 추상 지식 베이스 |
| `_simple_knowledge.py` | 기본 구현체 |
| `_document.py` | 문서 및 메타데이터 구조 |

### 리더 (문서 파서)

| 파일 | 역할 |
|------|------|
| `_reader/_text_reader.py` | 텍스트 파일 |
| `_reader/_pdf_reader.py` | PDF 파일 |
| `_reader/_word_reader.py` | DOCX 파일 |
| `_reader/_ppt_reader.py` | PowerPoint |
| `_reader/_excel_reader.py` | Excel 파일 |
| `_reader/_image_reader.py` | 이미지 파일 |

### 벡터 저장소

| 파일 | 역할 |
|------|------|
| `_store/_store_base.py` | 추상 벡터 DB 인터페이스 |
| `_store/_qdrant_store.py` | Qdrant 연동 |
| `_store/_milvuslite_store.py` | Milvus Lite 연동 |
| `_store/_mongodb_store.py` | MongoDB Atlas 연동 |

---

## 핵심 클래스

### `Document`

```python
class Document:
    id: str                       # 고유 문서 ID
    metadata: DocMetadata         # 소스, 청크 정보 등
    embedding: Embedding | None   # 벡터 표현
    score: float | None           # 검색 관련도 점수

class DocMetadata:
    content: str           # 문서 텍스트 내용
    source: str            # 원본 파일 경로/URL
    chunk_index: int       # 청크 인덱스
    total_chunks: int      # 전체 청크 수
    extra: dict            # 추가 메타데이터
```

---

### `KnowledgeBase`

```python
class KnowledgeBase:
    embedding_store: VDBStoreBase
    embedding_model: EmbeddingModelBase

    async def add_documents(documents: list[Document]) -> None

    async def retrieve(
        query: str,
        limit: int = 5,
        score_threshold: float | None = None
    ) -> list[Document]

    # 에이전트 툴로 노출
    async def retrieve_knowledge(
        query: str,
        limit: int = 5,
        score_threshold: float | None = None
    ) -> ToolResponse
```

---

### `ReaderBase`

```python
class ReaderBase:
    async def __call__(*args, **kwargs) -> list[Document]
    def get_doc_id(*args, **kwargs) -> str   # 중복 감지용 ID 생성
```

---

### `VDBStoreBase`

```python
class VDBStoreBase:
    async def add(documents: list[Document], **kwargs) -> None
    async def search(
        query_embedding: Embedding,
        limit: int,
        score_threshold: float | None
    ) -> list[Document]
    async def delete(*args, **kwargs) -> None
    def get_client() -> Any   # 내부 DB 클라이언트 접근
```

---

## 지원 벡터 데이터베이스

| 벡터 DB | 특징 | 설치 |
|---------|------|------|
| Qdrant | 빠른 근사 최근접 이웃 검색 | `pip install qdrant-client` |
| Milvus Lite | 경량, 임베디드 | `pip install pymilvus` |
| MongoDB Atlas | 클라우드, 벡터 검색 지원 | `pip install pymongo` |
| MySQL | SQL 기반 벡터 검색 | `pip install mysqlclient` |
| OceanBase | 알리바바 클라우드 DB | 별도 설치 |

---

## RAG 파이프라인

```
문서 수집
  ↓
ReaderBase.__call__()  →  list[Document]
  ↓
EmbeddingModelBase()   →  Document.embedding 생성
  ↓
VDBStoreBase.add()     →  벡터 DB 저장
  ↓
--- (검색 시) ---
  ↓
KnowledgeBase.retrieve(query)
  → EmbeddingModelBase(query)  →  query_embedding
  → VDBStoreBase.search()      →  유사 문서 반환
  ↓
에이전트에 관련 문서 제공
```

---

## 문서 청킹

대용량 문서는 자동으로 청크로 분할:

```python
reader = PDFReader(
    chunk_size=512,      # 청크당 토큰 수
    chunk_overlap=50,    # 청크 간 중복 토큰 수
)
documents = await reader("./research_paper.pdf")
# → [Document(chunk_index=0), Document(chunk_index=1), ...]
```

---

## 모듈 연동

```
RAG
  ← Agent        (retrieve_knowledge 툴 호출)
  → Embedding    (문서/쿼리 벡터화)
  → VDB Store    (벡터 저장/검색)
  ← ReActAgent   (쿼리 재작성 후 검색)
```
