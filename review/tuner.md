# Tuner 모듈 분석

> [← 목차로 돌아가기](./one.md)

## 개요

에이전트 프롬프트와 모델을 자동으로 최적화하는 튜닝 모듈. DSPy의 MIPROv2 알고리즘을 사용한 프롬프트 튜닝, Trinity-RFT를 활용한 강화학습 기반 파인튜닝, 그리고 모델 선택을 지원한다.

---

## 파일 구성

| 파일 | 역할 |
|------|------|
| `_tune.py` | 튜닝 진입점 함수 `tune()` |
| `_config.py` | 튜닝 설정 관리 |
| `_workflow.py` | 워크플로우 정의 |
| `_judge.py` | 평가 함수 (judge) |
| `_dataset.py` | 데이터셋 설정 |
| `_algorithm.py` | 최적화 알고리즘 |
| `prompt_tune/` | DSPy MIPROv2 프롬프트 최적화 |
| `model_selection/` | 모델 선택 로직 |

---

## 핵심 함수

### `tune()`

```python
async def tune(
    workflow_func: Callable,          # 평가할 에이전트 워크플로우
    judge_func: Callable,             # 결과 평가 함수
    train_dataset: DatasetConfig,     # 학습 데이터셋
    eval_dataset: DatasetConfig,      # 평가 데이터셋
    model: ModelConfig,               # 최적화 대상 모델
    auxiliary_models: list[ModelConfig] | None,
    algorithm: AlgorithmConfig,       # 최적화 알고리즘
    project_name: str,
    experiment_name: str,
    monitor_type: str | None,         # "langfuse", "arize" 등
    config_path: str | None,
) -> TuningResult
```

---

## 프롬프트 튜닝 (DSPy MIPROv2)

```python
from agentscope.tuner import tune

result = await tune(
    workflow_func=my_agent_workflow,
    judge_func=accuracy_judge,
    train_dataset=DatasetConfig(path="./train.jsonl"),
    eval_dataset=DatasetConfig(path="./eval.jsonl"),
    model=ModelConfig(name="gpt-4o-mini"),
    algorithm=AlgorithmConfig(
        type="miprov2",
        num_candidates=10,    # 프롬프트 후보 수
        max_bootstrapped_demos=3,
    ),
    project_name="my_project",
    experiment_name="prompt_opt_v1",
)

# 최적화된 프롬프트 저장 위치
print(result.best_prompt)
```

**MIPROv2 동작 원리**:
1. 여러 프롬프트 후보 생성
2. 학습 데이터로 각 후보 평가
3. 베이지안 최적화로 최적 프롬프트 선택
4. 평가 데이터로 최종 검증

---

## 모델 선택

```python
result = await tune(
    ...
    algorithm=AlgorithmConfig(
        type="model_selection",
        candidate_models=[
            "gpt-4o-mini",
            "claude-haiku-4-5",
            "gemini-1.5-flash",
        ],
        selection_metric="cost_efficiency",  # 비용 효율 기준 선택
    ),
)

print(result.best_model)  # 최적 모델명
```

---

## 강화학습 파인튜닝 (Trinity-RFT)

```python
result = await tune(
    ...
    algorithm=AlgorithmConfig(
        type="rl_finetuning",
        base_model="qwen2.5-7b",
        rl_algorithm="GRPO",     # Group Relative Policy Optimization
        num_epochs=3,
        learning_rate=1e-5,
    ),
)
```

Trinity-RFT와 연동하여 Ray 클러스터에서 분산 강화학습 실행.

---

## 평가 함수 (Judge)

```python
async def accuracy_judge(
    workflow_output: Any,
    ground_truth: Any,
    task: Task,
) -> float:
    """0.0 ~ 1.0 사이의 점수 반환"""
    if workflow_output == ground_truth:
        return 1.0
    return 0.0
```

---

## 데이터셋 설정

```python
class DatasetConfig:
    path: str           # 파일 경로 또는 HuggingFace 데이터셋 ID
    split: str          # "train", "validation", "test"
    format: str         # "jsonl", "csv", "hf"
    input_field: str    # 입력 필드명
    output_field: str   # 정답 필드명
```

---

## 모니터링

튜닝 과정을 외부 플랫폼으로 추적:

```python
result = await tune(
    ...
    monitor_type="langfuse",   # 또는 "arize", "wandb"
)
```

---

## 모듈 연동

```
Tuner
  ← Evaluate   (평가 메트릭으로 피드백)
  → Model      (최적화된 프롬프트/모델 설정 업데이트)
  → Tracing    (튜닝 과정 추적)
  ← Dataset    (HuggingFace datasets 연동)
```
