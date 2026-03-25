# 평가 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/evaluation/ace_bench/`

---

## ace_bench — ACE 벤치마크 평가

ACE-Bench(에이전트 능력 평가 벤치마크)를 Ray 분산 평가로 실행하는 예제.

**파일**: `main.py`

---

## 실제 코드

```python
# examples/evaluation/ace_bench/main.py

async def react_agent_solution(
    ace_task: Task,
    pre_hook: Callable,
) -> SolutionOutput:
    """ACE 태스크를 ReAct 에이전트로 풀기"""

    # 태스크에서 툴 동적 등록
    toolkit = Toolkit()
    for tool, json_schema in ace_task.metadata["tools"]:
        toolkit.register_tool_function(tool, json_schema=json_schema)

    agent = ReActAgent(
        name="Friday",
        sys_prompt="You are a helpful assistant named Friday. "
                   "Your target is to solve the given task with your tools.",
        model=DashScopeChatModel(
            api_key=os.environ.get("DASHSCOPE_API_KEY"),
            model_name="qwen3-max",
            stream=False,
        ),
        formatter=DashScopeChatFormatter(),
        toolkit=toolkit,
    )

    # pre_print 훅으로 에이전트 입력 메시지 로깅
    agent.register_instance_hook("pre_print", "save_logging", pre_hook)

    msg_input = Msg("user", ace_task.input, role="user")
    await agent.print(msg_input)  # pre_print 훅 트리거
    await agent(msg_input)

    # 툴 호출 궤적 수집
    traj = []
    for msg in await agent.memory.get_memory():
        traj.extend(msg.get_content_blocks(["tool_use", "tool_result"]))

    # 최종 상태 (폰/여행 시스템)
    phone: ACEPhone = ace_task.metadata["phone"]
    final_state = phone.get_current_state()

    return SolutionOutput(
        success=True,
        output=final_state,
        trajectory=traj,
    )


async def main():
    evaluator = RayEvaluator(
        name="ACEbench evaluation",
        benchmark=ACEBenchmark(data_dir=args.data_dir),
        n_repeat=1,
        storage=FileEvaluatorStorage(save_dir=args.result_dir),
        n_workers=args.n_workers,
    )
    await evaluator.run(react_agent_solution)
```

**실행 방법:**
```bash
python main.py \
    --data_dir ./ace_data \
    --result_dir ./results \
    --n_workers 4
```

---

## 평가 프레임워크 구성요소

| 컴포넌트 | 클래스 | 역할 |
|---------|--------|------|
| 벤치마크 | `ACEBenchmark` | 태스크 로딩 및 정답 관리 |
| 솔루션 | `react_agent_solution` | 태스크 해결 함수 |
| 출력 | `SolutionOutput` | `success`, `output`, `trajectory` |
| 평가자 | `RayEvaluator` | Ray 분산 실행 |
| 저장소 | `FileEvaluatorStorage` | 결과 JSON 파일 저장 |
| 툴 환경 | `ACEPhone` | 스마트폰/앱 시뮬레이션 |

---

## 핵심 패턴

**동적 툴 등록:**
```python
for tool, json_schema in ace_task.metadata["tools"]:
    toolkit.register_tool_function(tool, json_schema=json_schema)
```
- 각 태스크마다 서로 다른 툴 세트를 주입
- `json_schema` 직접 지정 — 독스트링 파싱 없이 스키마 지정 가능

**pre_print 훅으로 입력 로깅:**
```python
agent.register_instance_hook("pre_print", "save_logging", pre_hook)
await agent.print(msg_input)  # 훅 트리거
```

**툴 궤적 수집:**
```python
traj.extend(msg.get_content_blocks(["tool_use", "tool_result"]))
```
- 메모리에서 `tool_use`/`tool_result` 블록만 추출하여 평가 데이터로 활용

---

## GeneralEvaluator vs RayEvaluator

```python
# 로컬 디버깅용 (Ray 없이)
from agentscope.evaluate import GeneralEvaluator

evaluator = GeneralEvaluator(
    benchmark=ACEBenchmark(data_dir="./data"),
    n_repeat=1,
    n_workers=2,  # asyncio 병렬
)

# 분산 실행용 (Ray 필요)
evaluator = RayEvaluator(
    benchmark=ACEBenchmark(data_dir="./data"),
    n_workers=8,  # Ray 워커 수
)
```
