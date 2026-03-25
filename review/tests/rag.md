# RAG 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (3개)

| 파일 | 테스트 대상 |
|------|-----------|
| `rag_reader_test.py` | 문서 리더 (텍스트/PDF/Word/Excel/PPT) |
| `rag_store_test.py` | 벡터 스토어 CRUD |
| `rag_knowledge_test.py` | KnowledgeBase 통합 |

---

## rag_reader_test.py

다양한 형식의 문서 파싱 및 청킹 검증. 테스트 데이터 파일(`test.docx`, `test.pptx`, `test.xlsx`) 사용.

```python
class TestDocumentReaders(IsolatedAsyncioTestCase):

    async def test_text_reader_char_split(self):
        """텍스트 파일 - 문자 단위 청킹"""
        reader = TextReader(
            split_strategy="char",
            chunk_size=100,
            chunk_overlap=20,
        )
        docs = await reader("./test_data/sample.txt")

        self.assertGreater(len(docs), 0)
        # 각 청크가 chunk_size 이하인지 검증
        for doc in docs:
            self.assertLessEqual(len(doc.metadata.content), 120)  # overlap 포함

    async def test_text_reader_sentence_split(self):
        """텍스트 파일 - 문장 단위 청킹"""
        reader = TextReader(
            split_strategy="sentence",
            chunk_size=3,  # 3문장 단위
        )
        docs = await reader("./test_data/sample.txt")
        # 각 청크에 최대 3문장 포함되어야 함

    async def test_text_reader_paragraph_split(self):
        """텍스트 파일 - 단락 단위 청킹"""
        reader = TextReader(split_strategy="paragraph")
        docs = await reader("./test_data/sample.txt")

    async def test_pdf_reader(self):
        """PDF 파일 파싱"""
        reader = PDFReader(chunk_size=512)
        docs = await reader("./test.pdf")

        self.assertGreater(len(docs), 0)
        # 각 Document의 필수 필드 검증
        for doc in docs:
            self.assertIsNotNone(doc.id)
            self.assertIsNotNone(doc.metadata.content)
            self.assertIsNotNone(doc.metadata.source)
            self.assertGreaterEqual(doc.metadata.chunk_index, 0)

    async def test_word_reader(self):
        """Word 문서 (.docx) 파싱"""
        reader = WordReader()
        docs = await reader("./test.docx")
        self.assertGreater(len(docs), 0)

    async def test_excel_reader(self):
        """Excel 파일 (.xlsx) 파싱"""
        reader = ExcelReader()
        docs = await reader("./test.xlsx")
        # 각 행이 Document로 변환되어야 함
        self.assertGreater(len(docs), 0)

    async def test_ppt_reader(self):
        """PowerPoint (.pptx) 파싱"""
        reader = PowerPointReader()
        docs = await reader("./test.pptx")
        # 각 슬라이드가 Document로 변환
        self.assertGreater(len(docs), 0)

    async def test_doc_id_deduplication(self):
        """동일 파일은 같은 doc_id 생성 (중복 방지)"""
        reader = TextReader()
        id1 = reader.get_doc_id("./sample.txt", chunk_index=0)
        id2 = reader.get_doc_id("./sample.txt", chunk_index=0)
        self.assertEqual(id1, id2)
```

---

## rag_store_test.py

벡터 스토어 CRUD 및 유사도 검색 검증.

```python
class TestVectorStore(IsolatedAsyncioTestCase):

    async def asyncSetUp(self):
        # 인메모리 벡터 스토어 (테스트용)
        self.store = InMemoryVectorStore(embedding_dim=128)

    async def test_add_and_search(self):
        """문서 추가 후 유사도 검색"""
        docs = [
            Document(
                id="doc1",
                metadata=DocMetadata(content="파이썬 프로그래밍"),
                embedding=[0.1] * 128,
            ),
            Document(
                id="doc2",
                metadata=DocMetadata(content="자바 프로그래밍"),
                embedding=[0.2] * 128,
            ),
        ]
        await self.store.add(docs)

        results = await self.store.search(
            query_embedding=[0.1] * 128,
            limit=2,
        )
        self.assertEqual(len(results), 2)
        # 첫 번째 결과가 가장 유사한 문서
        self.assertEqual(results[0].id, "doc1")

    async def test_score_threshold(self):
        """점수 임계값 필터링"""
        results = await self.store.search(
            query_embedding=query_vec,
            limit=10,
            score_threshold=0.8,  # 80% 이상 유사도만 반환
        )
        for doc in results:
            self.assertGreaterEqual(doc.score, 0.8)

    async def test_delete(self):
        """문서 삭제"""
        await self.store.add([doc])
        await self.store.delete(["doc1"])

        results = await self.store.search(query_vec, limit=10)
        ids = [r.id for r in results]
        self.assertNotIn("doc1", ids)
```

---

## rag_knowledge_test.py

```python
class TestKnowledgeBase(IsolatedAsyncioTestCase):

    async def test_add_and_retrieve(self):
        """문서 추가 및 쿼리 검색 통합 테스트"""
        mock_embedding = MockEmbeddingModel()
        store = InMemoryVectorStore(embedding_dim=128)
        kb = KnowledgeBase(
            embedding_store=store,
            embedding_model=mock_embedding,
        )

        docs = [Document(id="1", metadata=DocMetadata(content="AgentScope 소개"))]
        await kb.add_documents(docs)

        results = await kb.retrieve("AgentScope란?", limit=3)
        self.assertGreater(len(results), 0)

    async def test_retrieve_knowledge_tool(self):
        """에이전트 툴로 노출되는 retrieve_knowledge 함수"""
        tool_response = await kb.retrieve_knowledge(
            query="AgentScope란?",
            limit=5,
        )
        self.assertIsInstance(tool_response, ToolResponse)
        self.assertGreater(len(tool_response.content), 0)
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
| 청크 크기 검증 | ✓ | — | — |
| 중복 ID 생성 | ✓ | — | — |
| 문서 추가 | — | ✓ | ✓ |
| 유사도 검색 | — | ✓ | ✓ |
| 점수 임계값 | — | ✓ | ✓ |
| 문서 삭제 | — | ✓ | — |
| 툴 인터페이스 | — | — | ✓ |
