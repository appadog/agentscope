# 튜너 예제 분석

> [← 예제 개요](./overview.md) | [← 요약](../summary.md)

## 위치: `examples/tuner/`

프롬프트 자동 최적화, 모델 선택, 모델 파인튜닝 3가지 최적화 예제.

---

## 1. prompt_tuning — 프롬프트 자동 최적화

**파일**: `example.py`

GSM8K 수학 데이터셋으로 ReAct 에이전트의 시스템 프롬프트를 DSPy MIPROv2로 최적화.

**실제 코드:**

```python
# examples/tuner/prompt_tuning/example.py
model = DashScopeChatModel(
    "qwen-flash",
    api_key=os.environ.get("DASHSCOPE_API_KEY", ""),
    max_tokens=512,
)


async def workflow(task: dict, system_prompt: str) -> WorkflowOutput:
    """최적화될 워크플로우 함수"""
    toolkit = Toolkit()
    toolkit.register_tool_function(execute_python_code)

    agent = ReActAgent(
        name="react_agent",
        sys_prompt=system_prompt,  # 이 프롬프트가 최적화됨
        model=model,
        formatter=OpenAIChatFormatter(),
        toolkit=toolkit,
        print_hint_msg=False,
    )
    agent.set_console_output_enabled(False)

    response = await agent.reply(
        msg=Msg("user", task["question"], role="user"),
    )
    return WorkflowOutput(response=response)


async def gsm8k_judge(task: dict, response: Msg) -> JudgeOutput:
    """정답 판별 함수 (보상 계산)"""
    from trinity.common.rewards.math_reward import MathBoxedRewardFn

    reward_fn = MathBoxedRewardFn()

    # GSM8K 정답 파싱 ("#### 42" → "42")
    truth = task["answer"]
    if isinstance(truth, str) and "####" in truth:
        truth = truth.split("####")[1].strip()

    result = response.get_text_content()
    reward_dict = reward_fn(response=result, truth=truth)

    return JudgeOutput(
        reward=sum(reward_dict.values()),
        metrics=reward_dict,
    )


# 프롬프트 튜닝 실행
init_prompt = (
    "You are an agent. Please solve the math problem given to you with python code. "
    "You should provide your output within \\boxed{{}}."
)

optimized_prompt = tune_prompt(
    workflow=workflow,
    init_system_prompt=init_prompt,
    judge_func=gsm8k_judge,
    train_dataset=DatasetConfig(path="train.parquet", name="", split=""),
    eval_dataset=DatasetConfig(path="test.parquet", name="", split=""),
    config=PromptTuneConfig(
        lm_model_name="dashscope/qwen3-max",
        optimization_level="medium",   # "light" | "medium" | "heavy"
    ),
)

print(f"Optimized prompt: {optimized_prompt}")
```

**`tune_prompt()` 파라미터:**
| 파라미터 | 설명 |
|---------|------|
| `workflow` | 최적화할 에이전트 워크플로우 함수 |
| `init_system_prompt` | 초기 시스템 프롬프트 |
| `judge_func` | 응답 품질 평가 함수 (보상 반환) |
| `train_dataset` | 훈련 데이터셋 설정 |
| `eval_dataset` | 평가 데이터셋 설정 |
| `config.optimization_level` | `"light"` / `"medium"` / `"heavy"` |

---

## 2. model_selection — 모델 선택

**파일**: `example_bleu.py`, `example_token_usage.py`

여러 모델 중 최적 모델을 자동 선택.

### example_bleu.py — BLEU 점수 기반

```python
from agentscope.tuner import select_model

best_model = await select_model(
    candidates=[
        DashScopeChatModel("qwen-max", ...),
        DashScopeChatModel("qwen-turbo", ...),
        DashScopeChatModel("qwen-plus", ...),
    ],
    workflow=translation_workflow,
    eval_dataset=DatasetConfig(path="eval.parquet"),
    metric="bleu",
)
```

### example_token_usage.py — 토큰 사용량 기반

```python
best_model = await select_model(
    candidates=[...],
    workflow=workflow,
    eval_dataset=eval_dataset,
    metric="token_usage",  # 비용 최소화
)
```

---

## 3. model_tuning — 모델 파인튜닝 (Agentic RL)

**파일**: `main.py`

Trinity-RFT 프레임워크를 사용한 강화학습 기반 모델 파인튜닝.

```python
from agentscope.tuner import tune

# 강화학습 파인튜닝
tuned_model = await tune(
    workflow=agent_workflow,
    judge_func=reward_function,
    train_dataset=DatasetConfig(path="train.parquet"),
    config=TuneConfig(
        base_model="qwen3-7b",
        training_method="grpo",  # Group Relative Policy Optimization
        n_epochs=3,
        batch_size=16,
    ),
)
```

---

## 세 가지 최적화 비교

| 방식 | 대상 | 비용 | 사용 시기 |
|------|------|------|---------|
| `tune_prompt()` | 시스템 프롬프트 | 낮음 (API 호출만) | 모델 고정, 프롬프트 개선 |
| `select_model()` | 모델 선택 | 낮음 | 여러 모델 중 최적 선택 |
| `tune()` | 모델 가중치 | 높음 (GPU 필요) | 특정 태스크 전용 모델 |
