# 메모리 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (4개)

| 파일 | 테스트 클래스 |
|------|-------------|
| `memory_test.py` | `ShortTermMemoryTest` |
| `memory_compression_test.py` | `MemoryCompressionTest` |
| `memory_reme_test.py` | ReMeShortTermMemory |
| `mem0_utils_test.py` | mem0ai 통합 유틸리티 |

---

## memory_test.py

**테스트 클래스:** `ShortTermMemoryTest(IsolatedAsyncioTestCase)`

세 가지 백엔드(`InMemoryMemory`, `AsyncSQLAlchemyMemory`, `RedisMemory`)에 동일한 테스트를 적용하는 헬퍼 메서드 구조.

**`_basic_tests()` — 공통 기본 동작:**

```python
async def _basic_tests(self, memory: MemoryBase):
    msgs = [
        Msg("user", "0", "user"),
        Msg("assistant", "2", "assistant"),
        Msg("system", "3", "system"),
    ]
    # ID 수동 설정 (삭제 테스트용)
    for i, msg in enumerate(msgs):
        msg.id = str(i)

    # 배치 추가
    await memory.add(msgs)
    self.assertEqual(await memory.size(), 3)

    # 전체 조회
    retrieved = await memory.get_memory()
    self.assertEqual(len(retrieved), 3)

    # ID 기반 삭제
    await memory.delete([msgs[0].id])
    self.assertEqual(await memory.size(), 2)

    # 전체 초기화
    await memory.clear()
    self.assertEqual(await memory.size(), 0)
```

**`_mark_tests()` — 마크 기반 필터링:**

```python
async def _mark_tests(self, memory: MemoryBase):
    msg1 = Msg("user", "task1 메시지", "user")
    msg2 = Msg("user", "task2 메시지", "user")
    msg3 = Msg("user", "마크 없는 메시지", "user")

    await memory.add([msg1], marks=["task_1"])
    await memory.add([msg2], marks=["task_2"])
    await memory.add([msg3])

    # 특정 마크만 조회
    result = await memory.get_memory(mark="task_1")
    self.assertEqual(len(result), 1)
    self.assertEqual(result[0].content, "task1 메시지")

    # 특정 마크 제외 조회
    result = await memory.get_memory(exclude_mark="task_2")
    self.assertEqual(len(result), 2)  # msg1, msg3

    # 마크 업데이트
    await memory.update_messages_mark(new_mark="updated", old_mark="task_1")
    result = await memory.get_memory(mark="updated")
    self.assertEqual(len(result), 1)
```

**`_multi_tenant_tests()` — 멀티 테넌트 격리:**

```python
async def _multi_tenant_tests(self, memory: MemoryBase):
    """서로 다른 tenant_id 간 데이터 격리 검증"""
    await memory.add([msg_a], tenant_id="user_A")
    await memory.add([msg_b], tenant_id="user_B")

    result_a = await memory.get_memory(tenant_id="user_A")
    result_b = await memory.get_memory(tenant_id="user_B")

    self.assertEqual(len(result_a), 1)
    self.assertEqual(len(result_b), 1)
    self.assertNotEqual(result_a[0].id, result_b[0].id)
```

**백엔드별 테스트:**

```python
async def test_in_memory_backend(self):
    memory = InMemoryMemory()
    await self._basic_tests(memory)
    await self._mark_tests(memory)

async def test_sqlalchemy_backend(self):
    memory = AsyncSQLAlchemyMemory(db_url="sqlite+aiosqlite:///:memory:")
    await self._basic_tests(memory)
    await self._mark_tests(memory)

async def test_redis_backend(self):
    import fakeredis.aioredis
    redis = fakeredis.aioredis.FakeRedis()
    memory = RedisMemory(redis_client=redis, session_id="test_session")
    await self._basic_tests(memory)
```

---

## memory_compression_test.py

**테스트 클래스:** `MemoryCompressionTest(IsolatedAsyncioTestCase)`

**목 클래스:**

```python
class MockChatModel(ChatModelBase):
    """압축 요약 응답을 반환하는 목 모델"""
    def __init__(self):
        super().__init__("mock_model", stream=False)
        self.received_messages = []
        self.call_count = 0

    async def __call__(self, messages: list[dict], **kwargs) -> ChatResponse:
        self.received_messages = messages
        self.call_count += 1
        return ChatResponse(
            content=[TextBlock(type="text", text="압축된 대화 요약")]
        )

class MockFormatter(FormatterBase):
    """단순 메시지 → dict 변환 포매터"""
    async def format(self, msgs, **kwargs) -> list[dict]:
        return [{"role": m.role, "content": str(m.content)} for m in msgs]

    def get_tool_schemas(self, toolkit=None):
        return []
```

**임계값 이하 — 압축 미실행:**

```python
async def test_no_compression_below_threshold(self):
    agent = ReActAgent(
        name="test",
        model=MockChatModel(),
        formatter=MockFormatter(),
        memory=InMemoryMemory(),
        compression_config=ReActAgent.CompressionConfig(
            enable=True,
            trigger_threshold=10000,  # 높은 임계값
            agent_token_counter=CharTokenCounter(),
            keep_recent=1,
        ),
    )

    msg = Msg(name="user", content="짧은 메시지", role="user")
    await agent.reply(msg)

    # 모델이 압축 요청을 받지 않았어야 함 (reply 1회만)
    self.assertEqual(agent._model.call_count, 1)
```

**임계값 초과 — 압축 실행:**

```python
async def test_compression_above_threshold(self):
    agent = ReActAgent(
        name="test",
        model=MockChatModel(),
        formatter=MockFormatter(),
        memory=InMemoryMemory(),
        compression_config=ReActAgent.CompressionConfig(
            enable=True,
            trigger_threshold=10,   # 낮은 임계값 (10자)
            agent_token_counter=CharTokenCounter(),
            keep_recent=1,
        ),
    )

    # 임계값을 초과하는 긴 메시지
    msg = Msg(name="user", content="매우 긴 메시지 " * 100, role="user")
    await agent.reply(msg)

    # 모델이 압축 요청을 받아야 함
    self.assertGreater(agent._model.call_count, 1)
    # 메모리에 요약이 저장되어 있어야 함
    self.assertIsNotNone(agent.memory._compressed_summary)
```

---

## 검증 항목 요약

| 항목 | InMemory | SQL | Redis | Compression |
|------|---------|-----|-------|-------------|
| add/delete/clear | ✓ | ✓ | ✓ | — |
| size() | ✓ | ✓ | ✓ | — |
| get_memory() | ✓ | ✓ | ✓ | — |
| 마크 필터링 | ✓ | ✓ | — | — |
| 마크 업데이트 | ✓ | ✓ | — | — |
| 멀티 테넌트 격리 | ✓ | ✓ | ✓ | — |
| 압축 임계값 미트리거 | — | — | — | ✓ |
| 압축 실행 및 요약 저장 | — | — | — | ✓ |
| keep_recent 보존 | — | — | — | ✓ |
