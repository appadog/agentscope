# 실시간 테스트 분석

> [← 테스트 개요](./overview.md) | [← 요약](../summary.md)

## 파일 목록 (4개)

| 파일 | 테스트 대상 |
|------|-----------|
| `realtime_event_test.py` | 실시간 이벤트 타입 변환 |
| `realtime_openai_test.py` | OpenAI Realtime API |
| `realtime_gemini_test.py` | Gemini Live API |
| `realtime_dashscope_test.py` | DashScope 실시간 API |

---

## realtime_event_test.py

모델별 이벤트를 AgentScope 공통 이벤트로 변환하는 로직 검증.

```python
class TestRealtimeEvents(unittest.TestCase):

    def test_model_response_created_event(self):
        """ModelResponseCreatedEvent 변환"""
        model_event = {
            "type": "response.created",
            "response": {"id": "resp_001", "status": "in_progress"},
        }
        server_event = ServerEvents.from_model_event(model_event)

        self.assertIsInstance(server_event, ResponseCreatedEvent)
        self.assertEqual(server_event.response_id, "resp_001")

    def test_audio_delta_event(self):
        """오디오 청크 이벤트 변환"""
        model_event = {
            "type": "response.audio.delta",
            "delta": base64.b64encode(b"audio_data").decode(),
        }
        server_event = ServerEvents.from_model_event(model_event)

        self.assertIsInstance(server_event, ResponseAudioDeltaEvent)
        self.assertEqual(server_event.delta, b"audio_data")

    def test_text_delta_event(self):
        """텍스트 스트리밍 이벤트 변환"""
        model_event = {
            "type": "response.text.delta",
            "delta": "안녕",
        }
        server_event = ServerEvents.from_model_event(model_event)
        self.assertIsInstance(server_event, ResponseTextDeltaEvent)

    def test_function_call_event(self):
        """툴 호출 이벤트 변환"""
        model_event = {
            "type": "response.function_call_arguments.done",
            "call_id": "call_001",
            "name": "search",
            "arguments": '{"query": "파이썬"}',
        }
        server_event = ServerEvents.from_model_event(model_event)
        self.assertIsInstance(server_event, ResponseFunctionCallEvent)
        self.assertEqual(server_event.name, "search")

    def test_response_done_event(self):
        """응답 완료 이벤트"""
        model_event = {"type": "response.done"}
        server_event = ServerEvents.from_model_event(model_event)
        self.assertIsInstance(server_event, ResponseDoneEvent)

    def test_session_update_event(self):
        """세션 설정 업데이트 이벤트"""
        ...
```

---

## realtime_openai_test.py

OpenAI Realtime API WebSocket 통신 검증.

```python
class TestOpenAIRealtimeModel(IsolatedAsyncioTestCase):

    async def test_connect_and_send_audio(self):
        """WebSocket 연결 후 오디오 데이터 전송"""
        with patch("websockets.connect") as mock_ws:
            mock_websocket = AsyncMock()
            mock_ws.return_value.__aenter__.return_value = mock_websocket

            model = OpenAIRealtimeModel(
                model_name="gpt-4o-realtime-preview",
                api_key="test_key",
            )

            outgoing_queue = asyncio.Queue()
            await model.connect(
                outgoing_queue=outgoing_queue,
                instructions="테스트 에이전트입니다.",
                tools=[],
            )

            # 오디오 블록 전송
            audio_block = AudioBlock(
                type="audio",
                source=Base64Source(
                    type="base64",
                    media_type="audio/pcm",
                    data=base64.b64encode(b"test_audio").decode(),
                ),
            )
            await model.send(audio_block)

            # WebSocket send가 호출됐는지 검증
            mock_websocket.send.assert_called()

    async def test_receive_audio_response(self):
        """서버로부터 오디오 응답 수신"""
        # WebSocket에서 오디오 이벤트 시뮬레이션
        ...

    async def test_disconnect(self):
        """정상 종료"""
        await model.disconnect()
        self.assertFalse(model.is_connected)
```

---

## 이벤트 타입 매핑 검증

| 모델 이벤트 타입 | AgentScope 이벤트 |
|---------------|-----------------|
| `response.created` | `ResponseCreatedEvent` |
| `response.audio.delta` | `ResponseAudioDeltaEvent` |
| `response.text.delta` | `ResponseTextDeltaEvent` |
| `response.function_call_arguments.done` | `ResponseFunctionCallEvent` |
| `response.done` | `ResponseDoneEvent` |
| `session.created` | `SessionCreatedEvent` |
| `input_audio_buffer.speech_started` | `SpeechStartedEvent` |
| `input_audio_buffer.speech_stopped` | `SpeechStoppedEvent` |

---

## 검증 항목 요약

| 항목 | event | openai | gemini | dashscope |
|------|-------|--------|--------|-----------|
| 이벤트 타입 변환 | ✓ | — | — | — |
| WebSocket 연결 | — | ✓ | ✓ | ✓ |
| 오디오 전송 | — | ✓ | ✓ | ✓ |
| 텍스트 전송 | — | ✓ | ✓ | ✓ |
| 응답 수신/파싱 | — | ✓ | ✓ | ✓ |
| 툴 호출 처리 | — | ✓ | ✓ | ✓ |
| 정상 종료 | — | ✓ | ✓ | ✓ |
