# 튜너 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/tuner/`

프롬프트 자동 최적화, 모델 선택, 모델 파인튜닝 3가지 최적화 예제.

---

## 1. prompt_tuning — 프롬프트 자동 최적화

**파일**: `example.py`

DSPy의 MIPROv2 알고리즘으로 시스템 프롬프트를 자동 최적화.

```python
from agentscope.tuner import tune_prompt, PromptTuneConfig

# 최적화할 워크플로우 정의
async def my_workflow(
    inputs: dict,
    system_prompt: str,     # 튜닝 대상 프롬프트
) -> WorkflowOutput:
    agent = ReActAgent(
        sys_prompt=system_prompt,
        model=DashScopeChatModel(model_name="qwen-max"),
        ...
    )
    response = await agent.reply(Msg(content=inputs["question"]))
    return WorkflowOutput(answer=response.content)

# 평가 함수
async def judge(
    output: WorkflowOutput,
    expected: dict,
) -> JudgeOutput:
    is_correct = output.answer.strip() == expected["answer"].strip()
    return JudgeOutput(score=1.0 if is_correct else 0.0)

# 프롬프트 튜닝 실행
result = await tune_prompt(
    workflow_func=my_workflow,
    judge_func=judge,
    train_dataset=DatasetConfig(path="./train.jsonl"),
    eval_dataset=DatasetConfig(path="./eval.jsonl"),
    config=PromptTuneConfig(
        num_candidates=10,          # 생성할 프롬프트 후보 수
        max_bootstrapped_demos=3,   # 최대 예제 수
        num_trials=20,              # 베이지안 최적화 시도 횟수
    ),
)

print(f"최적 프롬프트:\n{result.best_prompt}")
print(f"평가 점수: {result.best_score:.2%}")
```

**MIPROv2 동작 원리**:
1. 훈련 데이터에서 여러 프롬프트 후보 자동 생성
2. 각 후보를 훈련 데이터로 평가
3. 베이지안 최적화로 최고 성능 프롬프트 탐색
4. 평가 데이터로 최종 검증

---

## 2. model_selection — 모델 비교 선택

### BLEU 점수 기준 선택

**파일**: `example_bleu.py`

```python
from agentscope.tuner import select_model

result = await select_model(
    workflow_func=translation_workflow,
    judge_func=bleu_judge,
    eval_dataset=DatasetConfig(path="./eval_translation.jsonl"),
    candidate_models=[
        "gpt-4o-mini",
        "claude-haiku-4-5-20251001",
        "gemini-1.5-flash",
        "qwen-max",
    ],
)

print(f"최적 모델: {result.best_model}")
print(f"모델별 점수:")
for model, score in result.scores.items():
    print(f"  {model}: {score:.4f}")
```

### 토큰 사용량 기준 선택

**파일**: `example_token_usage.py`

```python
result = await select_model(
    ...
    selection_metric="cost_efficiency",  # 비용 효율 기준
    cost_weights={
        "gpt-4o-mini": 0.15,      # $/1M 토큰
        "claude-haiku": 0.25,
        "gemini-flash": 0.075,
    },
)
```

---

## 3. model_tuning — 모델 파인튜닝

**파일**: `main.py`

Trinity-RFT와 연동하여 강화학습 기반 파인튜닝 수행.

```python
from agentscope.tuner import tune

result = await tune(
    workflow_func=coding_workflow,
    judge_func=code_correctness_judge,
    train_dataset=DatasetConfig(
        path="bigcode/the-stack",  # HuggingFace 데이터셋
        split="train",
        input_field="prompt",
        output_field="solution",
    ),
    eval_dataset=DatasetConfig(...),
    model=ModelConfig(name="qwen2.5-7b"),
    algorithm=AlgorithmConfig(
        type="rl_finetuning",
        rl_algorithm="GRPO",
        num_epochs=3,
        learning_rate=1e-5,
        batch_size=32,
    ),
    project_name="code_assistant",
    monitor_type="wandb",
)
```

---

## 최적화 방법 비교

| 방법 | 시간 | 비용 | 효과 | 적합한 경우 |
|------|------|------|------|------------|
| 프롬프트 튜닝 | 수 분 | 낮음 | 중간 | 빠른 개선, 기존 모델 유지 |
| 모델 선택 | 수 분 | 중간 | 중간 | 최적 모델 탐색 |
| 파인튜닝 | 수 시간 | 높음 | 높음 | 특수 도메인, 최고 성능 필요 |
