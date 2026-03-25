# RAG 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (3개)

| 파일 | 테스트 클래스 |
|------|-------------|
| `rag_reader_test.py` | `TestDocumentReaders` |
| `rag_store_test.py` | 벡터 스토어 CRUD |
| `rag_knowledge_test.py` | `RAGKnowledgeTest` |

---

## rag_knowledge_test.py

**테스트 클래스:** `RAGKnowledgeTest(IsolatedAsyncioTestCase)`

**목 임베딩 모델:**

```python
class TestTextEmbedding(EmbeddingModelBase):
    """고정 임베딩을 반환하는 테스트용 임베딩 모델"""
    MODALITIES = ["text"]

    async def __call__(self, inputs: list[str], **kwargs) -> list[list[float]]:
        # 특정 텍스트에 미리 정해진 임베딩 반환
        result = []
        for text in inputs:
            if "doc_0" in text or "파이썬" in text:
                result.append([1.0, 0.0, 0.0])
            elif "doc_1" in text or "자바" in text:
                result.append([0.0, 1.0, 0.0])
            else:
                result.append([0.0, 0.0, 1.0])
        return result
```

**지식 베이스 통합 테스트:**

```python
async def test_simple_knowledge(self):
    knowledge = SimpleKnowledge(
        embedding_model=TestTextEmbedding(),
        embedding_store=QdrantStore(
            location=":memory:",          # 인메모리 Qdrant
            collection_name="test",
            dimensions=3,
        ),
    )

    # 사전 계산된 임베딩으로 직접 추가
    docs = [
        Document(
            embedding=[1.0, 0.0, 0.0],
            metadata=DocMetadata(
                content=TextBlock(type="text", text="파이썬 프로그래밍"),
                doc_id="doc_0",
                chunk_id=0,
                total_chunks=1,
            ),
        ),
        Document(
            embedding=[0.0, 1.0, 0.0],
            metadata=DocMetadata(
                content=TextBlock(type="text", text="자바 프로그래밍"),
                doc_id="doc_1",
                chunk_id=0,
                total_chunks=1,
            ),
        ),
    ]
    await knowledge.add_documents(docs)

    # 쿼리 → 유사도 검색
    results = await knowledge.retrieve(
        query="파이썬에 대해 알려줘",
        limit=1,
        score_threshold=0.5,
    )

    self.assertEqual(len(results), 1)
    self.assertIn("파이썬", str(results[0].metadata.content))
```

---

## rag_reader_test.py

**테스트 클래스:** `TestDocumentReaders(IsolatedAsyncioTestCase)`

실제 테스트 파일(`test.docx`, `test.pptx`, `test.xlsx`)을 사용.

```python
async def test_text_reader_char_split(self):
    reader = TextReader(
        split_strategy="char",
        chunk_size=100,
        chunk_overlap=20,
    )
    docs = await reader("./test_data/sample.txt")

    self.assertGreater(len(docs), 0)
    for doc in docs:
        self.assertIsNotNone(doc.id)
        self.assertIsNotNone(doc.metadata.content)

async def test_text_reader_sentence_split(self):
    reader = TextReader(split_strategy="sentence", chunk_size=3)
    docs = await reader("./test_data/sample.txt")
    # 각 청크에 최대 3문장 포함

async def test_pdf_reader(self):
    reader = PDFReader(chunk_size=512)
    docs = await reader("./test.pdf")
    for doc in docs:
        self.assertIsNotNone(doc.id)
        self.assertIsNotNone(doc.metadata.content)
        self.assertIsNotNone(doc.metadata.source)
        self.assertGreaterEqual(doc.metadata.chunk_index, 0)

async def test_word_reader(self):
    reader = WordReader()
    docs = await reader("./test.docx")
    self.assertGreater(len(docs), 0)

async def test_excel_reader(self):
    reader = ExcelReader()
    docs = await reader("./test.xlsx")
    self.assertGreater(len(docs), 0)

async def test_ppt_reader(self):
    reader = PowerPointReader()
    docs = await reader("./test.pptx")
    self.assertGreater(len(docs), 0)

async def test_doc_id_deduplication(self):
    """동일 파일·동일 청크 인덱스 → 동일 doc_id 생성"""
    reader = TextReader()
    id1 = reader.get_doc_id("./sample.txt", chunk_index=0)
    id2 = reader.get_doc_id("./sample.txt", chunk_index=0)
    self.assertEqual(id1, id2)

    # 다른 청크 인덱스 → 다른 doc_id
    id3 = reader.get_doc_id("./sample.txt", chunk_index=1)
    self.assertNotEqual(id1, id3)
```

---

## rag_store_test.py

벡터 스토어 CRUD 및 유사도 검색 검증.

```python
class TestVectorStore(IsolatedAsyncioTestCase):

    async def asyncSetUp(self):
        self.store = QdrantStore(
            location=":memory:",
            collection_name="test",
            dimensions=128,
        )

    async def test_add_and_search(self):
        docs = [
            Document(
                embedding=[0.1] * 128,
                metadata=DocMetadata(
                    content=TextBlock(type="text", text="파이썬"),
                    doc_id="doc1",
                    chunk_id=0,
                    total_chunks=1,
                ),
            ),
            Document(
                embedding=[0.9] * 128,
                metadata=DocMetadata(
                    content=TextBlock(type="text", text="자바"),
                    doc_id="doc2",
                    chunk_id=0,
                    total_chunks=1,
                ),
            ),
        ]
        await self.store.add(docs)

        results = await self.store.search(
            query_embedding=[0.1] * 128,
            limit=2,
        )
        self.assertEqual(len(results), 2)
        # 첫 번째 결과가 더 높은 유사도
        self.assertGreater(results[0].score, results[1].score)

    async def test_score_threshold(self):
        results = await self.store.search(
            query_embedding=query_vec,
            limit=10,
            score_threshold=0.8,
        )
        for doc in results:
            self.assertGreaterEqual(doc.score, 0.8)

    async def test_delete(self):
        await self.store.add([doc])
        await self.store.delete(["doc1"])
        results = await self.store.search(query_vec, limit=10)
        ids = [r.metadata.doc_id for r in results]
        self.assertNotIn("doc1", ids)
```

---

## 검증 항목 요약

| 항목 | reader | store | knowledge |
|------|--------|-------|-----------|
| 텍스트 파일 파싱 | ✓ | — | — |
| PDF 파싱 | ✓ | — | — |
| Word 파싱 | ✓ | — | — |
| Excel 파싱 | ✓ | — | — |
| PPT 파싱 | ✓ | — | — |
| 청크 전략 (char/sentence/paragraph) | ✓ | — | — |
| 중복 doc_id 방지 | ✓ | — | — |
| 문서 추가 | — | ✓ | ✓ |
| 유사도 검색 | — | ✓ | ✓ |
| 점수 임계값 필터링 | — | ✓ | ✓ |
| 문서 삭제 | — | ✓ | — |
| 인메모리 벡터 DB 사용 | — | ✓ | ✓ |
