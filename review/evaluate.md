# Evaluate 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

에이전트와 모델의 성능을 측정하는 평가 프레임워크. 메트릭 정의, 벤치마크 태스크 관리, 분산 평가 실행, 결과 집계를 체계적으로 지원하며, ACE-Bench가 기본 벤치마크로 포함되어 있다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_metric_base.py` | 평가 메트릭 기반 클래스 |
| `_benchmark_base.py` | 벤치마크 정의 |
| `_evaluator/_evaluator_base.py` | 평가 실행기 추상 클래스 |
| `_evaluator/_general_evaluator.py` | 일반 평가 실행기 |
| `_evaluator/_ray_evaluator.py` | Ray 분산 평가 실행기 |
| `_evaluator_storage/` | 결과 저장 백엔드 |
| `_ace_benchmark/` | ACE (Agent Conversation Exchange) 벤치마크 |

---

## 핵심 클래스

### `MetricBase`

```python
class MetricBase:
    name: str
    metric_type: Literal["CATEGORY", "NUMERICAL"]
    description: str
    categories: list[str] | None   # CATEGORY 타입인 경우

    async def __call__(*args, **kwargs) -> MetricResult
```

---

### `MetricResult`

```python
class MetricResult:
    name: str
    result: str | float     # CATEGORY: 카테고리명, NUMERICAL: 수치
    created_at: datetime
    message: str | None     # 평가 근거
    metadata: dict
```

---

### `BenchmarkBase`

```python
class BenchmarkBase:
    name: str
    description: str

    async def get_tasks() -> list[Task]
```

---

### `Task`와 `SolutionOutput`

```python
class Task:
    id: str
    metadata: dict   # 태스크별 추가 정보

class SolutionOutput:
    task_id: str
    output: Any      # 에이전트의 답변
    metadata: dict
```

---

### `EvaluatorBase`

```python
class EvaluatorBase:
    name: str
    benchmark: BenchmarkBase
    n_repeat: int       # 반복 실행 횟수
    storage: StorageBase

    async def run(solution: Callable) -> list[MetricResult]
    async def aggregate() -> dict    # 결과 집계 (평균, 표준편차 등)
```

---

## 분산 평가 (Ray)

```python
from agentscope.evaluate import RayEvaluator

evaluator = RayEvaluator(
    name="react_agent_eval",
    benchmark=ace_benchmark,
    n_repeat=3,
    storage=JsonStorage("./results/"),
    num_workers=8,    # Ray 병렬 워커 수
)

# 평가할 에이전트 솔루션 정의
async def my_solution(task: Task) -> SolutionOutput:
    response = await agent.reply(Msg(..., content=task.metadata["question"]))
    return SolutionOutput(task_id=task.id, output=response.content)

# 평가 실행
results = await evaluator.run(my_solution)
summary = await evaluator.aggregate()
```

---

## ACE-Bench

AgentScope에 포함된 기본 벤치마크:

```
_ace_benchmark/
  ├─ tasks/       # 평가 태스크 데이터셋
  ├─ metrics/     # ACE 전용 메트릭
  └─ _ace_benchmark.py
```

- **목적**: 에이전트의 대화 능력, 툴 사용, 추론 능력 종합 평가
- **태스크 유형**: QA, 멀티턴 대화, 툴 호출, 계획 수립

---

## 메트릭 예시

```python
# 카테고리 메트릭 (정확도)
class AccuracyMetric(MetricBase):
    metric_type = "CATEGORY"
    categories = ["correct", "incorrect", "partial"]

    async def __call__(
        prediction: str,
        ground_truth: str
    ) -> MetricResult:
        ...

# 수치 메트릭 (BLEU 점수)
class BLEUMetric(MetricBase):
    metric_type = "NUMERICAL"

    async def __call__(
        prediction: str,
        reference: str
    ) -> MetricResult:
        score = calculate_bleu(prediction, reference)
        return MetricResult(name=self.name, result=score)
```

---

## 결과 저장 백엔드

| 백엔드 | 파일 |
|--------|------|
| JSON | `_json_storage.py` |
| SQLite | `_sqlite_storage.py` |

---

## 모듈 연동

```
Evaluate
  ← Agent      (평가 대상 솔루션)
  → Tuner      (평가 결과 → 튜닝 피드백)
  ← Benchmark  (태스크 제공)
  → Storage    (결과 저장)
```
