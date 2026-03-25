# 기타 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (17개)

| 카테고리 | 파일 | 테스트 대상 |
|----------|------|-----------|
| TTS | `tts_openai_test.py` | OpenAI TTS API |
| TTS | `tts_gemini_test.py` | Gemini TTS API |
| TTS | `tts_dashscope_test.py` | DashScope TTS |
| TTS | `tts_dashscope_cosyvoice_test.py` | DashScope CosyVoice |
| MCP | `mcp_sse_client_test.py` | SSE 기반 MCP 클라이언트 |
| MCP | `mcp_streamable_http_client_test.py` | Streamable HTTP MCP |
| 토큰 | `token_openai_test.py` | OpenAI 토크나이저 |
| 토큰 | `token_anthropic_test.py` | Anthropic 토크나이저 |
| 토큰 | `token_char_test.py` | 문자 기반 토큰 카운터 |
| 세션 | `session_test.py` | JSON/Redis 세션 |
| 파이프라인 | `pipeline_test.py` | 순차/팬아웃 파이프라인 |
| 설정 | `config_test.py` | 컨텍스트 변수 격리 |
| 임베딩 | `embedding_cache_test.py` | 임베딩 캐시 |
| 평가 | `evaluation_test.py` | 벤치마크 평가 프레임워크 |
| 플랜 | `plan_test.py` | 플랜 관리 |
| 튜너 | `tuner_test.py` | 파라미터 튜닝 |
| 사용자 입력 | `user_input_test.py` | 사용자 입력 처리 |

---

## TTS 테스트

```python
class TestOpenAITTS(IsolatedAsyncioTestCase):

    async def test_synthesize(self):
        """전체 텍스트 한 번에 음성 합성"""
        with patch("openai.AsyncOpenAI") as mock:
            mock_client = AsyncMock()
            mock_client.audio.speech.create.return_value = mock_audio_response

            tts = OpenAITTSModel(model_name="tts-1", voice="alloy")
            response = await tts.synthesize(
                Msg(name="assistant", content="안녕하세요", role="assistant")
            )

            self.assertIsInstance(response.audio, AudioBlock)

    async def test_streaming_push(self):
        """스트리밍 방식으로 텍스트 청크 순차 전송"""
        async with tts:
            for chunk in ["안녕", "하세", "요"]:
                response = await tts.push(
                    Msg(content=chunk, ...)
                )
```

---

## MCP 테스트

```python
class TestMCPSSEClient(IsolatedAsyncioTestCase):

    async def test_list_tools(self):
        """MCP 서버에서 툴 목록 조회"""
        with mock_sse_server(tools=["tool_a", "tool_b"]):
            client = HttpStatefulClient(name="test", url="http://localhost:8080/sse")
            async with client:
                tools = await client.list_tools()
                self.assertEqual(len(tools), 2)

    async def test_call_tool(self):
        """MCP 툴 호출 및 결과 수신"""
        with mock_sse_server():
            client = HttpStatefulClient(...)
            async with client:
                func = await client.get_callable_function("add")
                result = await func(a=1, b=2)
                self.assertIn("3", str(result))

    async def test_content_type_conversion(self):
        """MCP 응답 콘텐츠 → AgentScope 블록 변환"""
        mcp_content = [
            {"type": "text", "text": "결과"},
            {"type": "image", "data": "base64_data", "mimeType": "image/png"},
        ]
        blocks = client._convert_mcp_content_to_as_blocks(mcp_content)
        self.assertIsInstance(blocks[0], TextBlock)
        self.assertIsInstance(blocks[1], ImageBlock)
```

---

## 토큰 테스트

```python
class TestOpenAITokenCounter(unittest.TestCase):

    def test_simple_message_count(self):
        """단순 텍스트 메시지 토큰 수 계산"""
        msgs = [{"role": "user", "content": "hello world"}]
        count = count_tokens(msgs, model="gpt-4o")
        self.assertIsInstance(count, int)
        self.assertGreater(count, 0)

    def test_multimodal_message_count(self):
        """이미지 포함 메시지 토큰 수 계산"""
        # 1000+ 개의 다양한 메시지 조합으로 검증

    def test_tool_use_token_count(self):
        """툴 정의 및 호출 토큰 수 계산"""
        ...
```

---

## 파이프라인 테스트

```python
class TestPipeline(IsolatedAsyncioTestCase):

    async def test_sequential_pipeline(self):
        """순차 실행: A → B → C"""
        class AddAgent(AgentBase):
            async def reply(self, msg):
                return Msg(content=msg.content + "+1")

        agents = [AddAgent(), AddAgent(), AddAgent()]
        result = await sequential_pipeline(agents, Msg(content="0"))
        self.assertEqual(result.content, "0+1+1+1")

    async def test_fanout_parallel(self):
        """팬아웃 병렬 실행"""
        results = await fanout_pipeline(
            agents=[agent1, agent2, agent3],
            msg=initial_msg,
            sequential=False,
        )
        self.assertEqual(len(results), 3)

    async def test_stream_printing_with_error(self):
        """스트리밍 중 에러 처리"""
        class ErrorAgent(AgentBase):
            async def reply(self, msg):
                raise RuntimeError("에러 발생")

        with self.assertRaises(RuntimeError):
            async for _ in stream_printing_messages(
                agents=[ErrorAgent()],
                coroutine_task=agent(msg),
            ):
                pass
```

---

## 임베딩 캐시 테스트

```python
class TestFileEmbeddingCache(unittest.TestCase):

    def test_store_and_retrieve(self):
        """캐시 저장 및 조회"""
        cache = FileEmbeddingCache(cache_dir=self.temp_dir)
        embedding = [0.1, 0.2, 0.3]

        cache.store("hello world", embedding)
        retrieved = cache.retrieve("hello world")

        self.assertEqual(retrieved, embedding)

    def test_lru_eviction(self):
        """최대 파일 수 초과 시 가장 오래된 캐시 제거"""
        cache = FileEmbeddingCache(
            cache_dir=self.temp_dir,
            max_file_number=3,
        )
        # 4개 저장 → 첫 번째 항목 자동 제거
        for i in range(4):
            cache.store(f"text_{i}", [float(i)] * 10)

        self.assertIsNone(cache.retrieve("text_0"))  # 제거됨
        self.assertIsNotNone(cache.retrieve("text_3"))  # 최신

    def test_max_cache_size(self):
        """전체 캐시 크기 제한"""
        cache = FileEmbeddingCache(
            cache_dir=self.temp_dir,
            max_cache_size_mb=1,  # 1MB 제한
        )
        ...
```

---

## 평가 테스트

```python
class TestEvaluationFramework(IsolatedAsyncioTestCase):

    async def test_general_evaluator(self):
        """GeneralEvaluator로 벤치마크 평가"""
        benchmark = ToyBenchmark()  # 테스트용 간단한 벤치마크
        evaluator = GeneralEvaluator(
            benchmark=benchmark,
            n_repeat=2,
            n_workers=2,
        )
        results = await evaluator.run(simple_solution)
        self.assertGreater(len(results), 0)

    async def test_ray_evaluator(self):
        """Ray 분산 평가"""
        ray.init(num_cpus=2)
        try:
            evaluator = RayEvaluator(
                benchmark=benchmark,
                n_workers=2,
            )
            results = await evaluator.run(solution)
        finally:
            ray.shutdown()
```

---

## 설정 테스트

```python
class TestConfigContextVars(IsolatedAsyncioTestCase):

    async def test_context_var_isolation_in_tasks(self):
        """asyncio Task 간 컨텍스트 변수 격리"""
        run_id.set("task_A")

        async def check_in_task():
            self.assertEqual(run_id.get(), "task_B")

        task = asyncio.create_task(check_in_task())
        run_id.set("task_B")
        await task

    def test_context_var_isolation_in_threads(self):
        """Thread 간 컨텍스트 변수 격리"""
        import threading

        def thread_func():
            run_id.set("thread_1")
            time.sleep(0.1)
            # 다른 스레드의 set이 영향을 주면 안 됨
            self.assertEqual(run_id.get(), "thread_1")

        t = threading.Thread(target=thread_func)
        run_id.set("main_thread")
        t.start()
        t.join()
        self.assertEqual(run_id.get(), "main_thread")
```
