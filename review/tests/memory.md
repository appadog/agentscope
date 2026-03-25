# 메모리 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (4개)

| 파일 | 테스트 대상 |
|------|-----------|
| `memory_test.py` | InMemoryMemory, SQLAlchemyMemory, RedisMemory |
| `memory_compression_test.py` | 메모리 압축 파이프라인 |
| `memory_reme_test.py` | ReMeShortTermMemory |
| `mem0_utils_test.py` | mem0ai 통합 유틸리티 |

---

## memory_test.py

세 가지 백엔드의 공통 동작을 `_basic_tests` 헬퍼 메서드로 통합 검증.

```python
class TestMemory(IsolatedAsyncioTestCase):

    async def _basic_tests(self, memory: MemoryBase):
        """모든 백엔드에 공통으로 적용되는 기본 테스트"""

        # 메시지 추가
        msg1 = Msg(name="user", content="첫 번째 메시지", role="user")
        msg2 = Msg(name="assistant", content="응답", role="assistant")
        await memory.add([msg1, msg2])

        # 크기 확인
        size = await memory.size()
        self.assertEqual(size, 2)

        # 전체 조회
        msgs = await memory.get_memory()
        self.assertEqual(len(msgs), 2)

        # 특정 메시지 삭제
        await memory.delete([msg1.id])
        self.assertEqual(await memory.size(), 1)

        # 전체 초기화
        await memory.clear()
        self.assertEqual(await memory.size(), 0)

    async def test_in_memory_backend(self):
        memory = InMemoryMemory()
        await self._basic_tests(memory)

    async def test_sqlalchemy_backend(self):
        memory = AsyncSQLAlchemyMemory(
            db_url="sqlite+aiosqlite:///:memory:"
        )
        await self._basic_tests(memory)

    async def test_redis_backend(self):
        # fakeredis로 실제 Redis 없이 테스트
        import fakeredis.aioredis
        redis = fakeredis.aioredis.FakeRedis()
        memory = RedisMemory(redis_client=redis)
        await self._basic_tests(memory)

    async def test_mark_filtering(self):
        """마크 기반 선택적 조회"""
        memory = InMemoryMemory()

        await memory.add([msg1], marks=["task_1"])
        await memory.add([msg2], marks=["task_2"])
        await memory.add([msg3])  # 마크 없음

        # task_1 마크만 조회
        result = await memory.get_memory(mark="task_1")
        self.assertEqual(len(result), 1)

        # task_2 마크 제외 조회
        result = await memory.get_memory(exclude_mark="task_2")
        self.assertEqual(len(result), 2)  # msg1, msg3

    async def test_compressed_summary(self):
        """압축 요약 prepend 동작"""
        memory = InMemoryMemory()
        memory._compressed_summary = "이전 대화 요약..."

        msgs = await memory.get_memory(prepend_summary=True)
        # 첫 번째 메시지가 요약이어야 함
        self.assertEqual(msgs[0].content, "이전 대화 요약...")

    async def test_update_mark(self):
        """메시지 마크 업데이트"""
        await memory.add([msg], marks=["old_mark"])
        await memory.update_messages_mark(
            new_mark="new_mark",
            old_mark="old_mark",
        )
        result = await memory.get_memory(mark="new_mark")
        self.assertEqual(len(result), 1)
```

---

## memory_compression_test.py

메모리 압축 파이프라인 검증.

```python
class TestMemoryCompression(IsolatedAsyncioTestCase):

    async def test_compression_triggered(self):
        """토큰 임계값 초과 시 압축 자동 실행"""
        mock_model = MockChatModel(
            summary_response="대화 요약: 사용자가 파이썬 질문을 했음."
        )

        memory = InMemoryMemory()
        config = CompressionConfig(
            trigger_threshold=100,  # 100 토큰 초과 시 압축
            keep_recent=2,          # 최근 2개 메시지 보존
        )

        # 100 토큰 이상의 메시지들 추가
        for i in range(10):
            await memory.add([
                Msg(name="user", content=f"긴 메시지 {i} " * 20, role="user")
            ])

        await memory.compress(model=mock_model, config=config)

        # 압축 후 최근 2개만 남아야 함
        msgs = await memory.get_memory(prepend_summary=False)
        self.assertEqual(len(msgs), 2)

        # 요약이 저장되어 있어야 함
        self.assertIsNotNone(memory._compressed_summary)
```

---

## memory_reme_test.py

ReMe 단기 메모리 검증.

```python
async def test_reme_add_and_retrieve(self):
    """ReME 메모리의 구조화된 저장 및 검색"""
    memory = ReMeShortTermMemory(
        model=MockChatModel(...),
        max_tokens=500,
    )

    await memory.add([
        Msg(name="user", content="내 이름은 김철수야", role="user"),
        Msg(name="assistant", content="알겠습니다, 김철수님!", role="assistant"),
    ])

    # 개인 정보 검색
    retrieved = await memory.get_memory()
    self.assertIn("김철수", str(retrieved))
```

---

## 검증 항목 요약

| 항목 | InMemory | SQL | Redis | ReMe | mem0 |
|------|---------|-----|-------|------|------|
| add/delete/clear | ✓ | ✓ | ✓ | ✓ | — |
| size() | ✓ | ✓ | ✓ | ✓ | — |
| get_memory() | ✓ | ✓ | ✓ | ✓ | — |
| 마크 필터링 | ✓ | ✓ | ✓ | — | — |
| 압축 트리거 | ✓ | — | — | ✓ | — |
| 요약 prepend | ✓ | ✓ | ✓ | — | — |
| 세션 격리 | ✓ | ✓ | ✓ | — | — |
