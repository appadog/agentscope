# 평가 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/evaluation/ace_bench/`

---

## ace_bench — ACE 벤치마크 평가

**파일**: `main.py`

**목적**: `ACEBenchmark`와 `RayEvaluator`를 사용하여 ReAct 에이전트를 분산 환경에서 체계적으로 평가하는 예제.

---

## 핵심 코드

```python
from agentscope.evaluate import (
    ACEBenchmark,
    RayEvaluator,
    FileEvaluatorStorage,
)

# 벤치마크 정의
benchmark = ACEBenchmark()

# 결과 저장소
storage = FileEvaluatorStorage(save_dir="./results/ace_bench/")

# Ray 분산 평가 실행기
evaluator = RayEvaluator(
    name="react_agent_eval",
    benchmark=benchmark,
    n_repeat=3,         # 각 태스크 3회 반복 실행
    n_workers=8,        # Ray 병렬 워커 수
    storage=storage,
)

# 평가할 솔루션 정의 (에이전트 생성 + 응답 수집)
async def solution(task: Task) -> SolutionOutput:
    agent = ReActAgent(
        name="test_agent",
        model=DashScopeChatModel(model_name="qwen-max"),
        toolkit=Toolkit(),
        memory=InMemoryMemory(),
    )

    response = await agent.reply(
        Msg(name="user", content=task.metadata["question"], role="user")
    )

    return SolutionOutput(
        task_id=task.id,
        output=response.content,
        metadata={"trajectory": agent.memory.get_all()},
    )

# 평가 실행
results = await evaluator.run(solution)

# 결과 집계
summary = await evaluator.aggregate()
print(f"정확도: {summary['accuracy']:.2%}")
print(f"평균 응답 시간: {summary['avg_time']:.2f}s")
```

---

## 평가 흐름

```
ACEBenchmark.get_tasks()
  → [Task1, Task2, ..., TaskN]
        ↓
RayEvaluator.run(solution)
  → Ray 클러스터에서 병렬 실행
  → n_workers 개의 워커가 태스크 분배 처리
  → n_repeat 회 반복으로 안정성 측정
        ↓
MetricBase.__call__(output, ground_truth)
  → MetricResult (점수, 근거)
        ↓
FileEvaluatorStorage.save()
  → ./results/ace_bench/
        ↓
evaluator.aggregate()
  → 전체 성능 요약 (평균, 표준편차)
```

---

## 사전 훅 (Pre-hook) 활용

```python
# 평가 전 로깅 훅 등록
async def logging_pre_hook(agent, *args, **kwargs):
    logger.info(f"태스크 시작: {args[0].content[:50]}")
    return args, kwargs

agent.pre_reply.append(logging_pre_hook)
```

---

## ACEBenchmark 태스크 유형

| 유형 | 설명 |
|------|------|
| QA | 단답형 질의응답 |
| 멀티턴 대화 | 여러 턴에 걸친 대화 능력 |
| 툴 호출 | 적절한 툴 선택 및 사용 |
| 계획 수립 | 복잡한 태스크 분해 능력 |

---

## Ray 설정

```python
import ray

ray.init(
    num_cpus=8,
    runtime_env={
        "pip": ["agentscope[evaluate]"],
    },
)

# 평가 완료 후
ray.shutdown()
```

분산 환경에서 수백 개의 태스크를 동시에 평가 가능.
