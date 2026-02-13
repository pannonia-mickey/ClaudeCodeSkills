# LangChain Evaluation Guide

Comprehensive reference for evaluating LangChain applications using LangSmith evaluators, custom evaluators, and systematic quality measurement.

---

## LangSmith Evaluation Setup

### Prerequisites

```python
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls__..."
os.environ["LANGCHAIN_PROJECT"] = "evaluation"

from langsmith import Client
client = Client()
```

### Creating Evaluation Datasets

```python
# Create dataset
dataset = client.create_dataset(
    dataset_name="rag-qa-golden",
    description="Golden QA pairs for RAG pipeline evaluation",
)

# Add examples
examples = [
    {
        "inputs": {"question": "What is LCEL?"},
        "outputs": {"answer": "LCEL (LangChain Expression Language) is a declarative way to compose chains using the pipe operator."},
    },
    {
        "inputs": {"question": "How do retrievers work?"},
        "outputs": {"answer": "Retrievers fetch relevant documents from a vector store based on semantic similarity to the query."},
    },
    {
        "inputs": {"question": "What is a ReAct agent?"},
        "outputs": {"answer": "A ReAct agent interleaves reasoning and action steps, using tools to gather information before generating a final answer."},
    },
]

for ex in examples:
    client.create_example(
        inputs=ex["inputs"],
        outputs=ex["outputs"],
        dataset_id=dataset.id,
    )
```

### Uploading from CSV or DataFrame

```python
import pandas as pd

df = pd.read_csv("eval_data.csv")
dataset = client.upload_dataframe(
    df=df,
    name="qa-from-csv",
    input_keys=["question", "context"],
    output_keys=["expected_answer"],
)
```

---

## Built-in LangSmith Evaluators

### QA Correctness

Evaluates whether the prediction answers the question correctly based on the reference answer:

```python
from langsmith.evaluation import evaluate

results = evaluate(
    lambda inputs: {"output": chain.invoke(inputs["question"])},
    data="rag-qa-golden",
    evaluators=["qa"],
    experiment_prefix="v1-qa-eval",
)
```

### Chain-of-Thought QA

Uses chain-of-thought reasoning for more nuanced correctness evaluation:

```python
results = evaluate(
    chain_fn,
    data="rag-qa-golden",
    evaluators=["cot_qa"],
    experiment_prefix="v1-cot-eval",
)
```

### Context QA

Evaluates whether the answer is faithful to the provided context:

```python
def chain_with_context(inputs):
    result = rag_chain.invoke(inputs["question"])
    return {"output": result, "context": inputs.get("context", "")}

results = evaluate(
    chain_with_context,
    data="rag-with-context",
    evaluators=["context_qa"],
    experiment_prefix="v1-context-eval",
)
```

---

## Custom Evaluators

### Basic Custom Evaluator

```python
from langsmith.evaluation import EvaluationResult, run_evaluator

@run_evaluator
def length_check(run, example) -> EvaluationResult:
    """Check that the response is within acceptable length bounds."""
    prediction = run.outputs.get("output", "")
    word_count = len(prediction.split())

    if word_count < 10:
        score = 0.0
        comment = f"Too short: {word_count} words"
    elif word_count > 500:
        score = 0.5
        comment = f"Too long: {word_count} words"
    else:
        score = 1.0
        comment = f"Acceptable length: {word_count} words"

    return EvaluationResult(key="length_check", score=score, comment=comment)
```

### LLM-as-Judge Evaluator

```python
from langchain_openai import ChatOpenAI

@run_evaluator
def llm_judge_relevance(run, example) -> EvaluationResult:
    """Use an LLM to judge answer relevance to the question."""
    question = example.inputs.get("question", "")
    prediction = run.outputs.get("output", "")

    judge = ChatOpenAI(model="gpt-4o-mini", temperature=0)

    judgment = judge.invoke(
        f"Rate the relevance of this answer to the question on a scale of 0-10.\n\n"
        f"Question: {question}\n"
        f"Answer: {prediction}\n\n"
        f"Score (0-10):"
    )

    try:
        score = float(judgment.content.strip().split()[0]) / 10.0
    except (ValueError, IndexError):
        score = 0.5

    return EvaluationResult(
        key="relevance",
        score=score,
        comment=judgment.content,
    )
```

### Structured LLM Judge

```python
from pydantic import BaseModel, Field

class QualityJudgment(BaseModel):
    faithfulness: float = Field(ge=0, le=1, description="Is the answer grounded in the context?")
    relevance: float = Field(ge=0, le=1, description="Does the answer address the question?")
    coherence: float = Field(ge=0, le=1, description="Is the answer well-structured and clear?")
    reasoning: str = Field(description="Explanation for the scores")

judge_llm = ChatOpenAI(model="gpt-4o").with_structured_output(QualityJudgment)

@run_evaluator
def multi_aspect_judge(run, example) -> list[EvaluationResult]:
    """Evaluate multiple quality aspects in a single LLM call."""
    question = example.inputs.get("question", "")
    context = example.inputs.get("context", "")
    prediction = run.outputs.get("output", "")
    reference = example.outputs.get("answer", "")

    judgment = judge_llm.invoke(
        f"Evaluate the quality of this answer.\n\n"
        f"Question: {question}\n"
        f"Context: {context}\n"
        f"Reference answer: {reference}\n"
        f"Predicted answer: {prediction}"
    )

    return [
        EvaluationResult(key="faithfulness", score=judgment.faithfulness, comment=judgment.reasoning),
        EvaluationResult(key="relevance", score=judgment.relevance, comment=judgment.reasoning),
        EvaluationResult(key="coherence", score=judgment.coherence, comment=judgment.reasoning),
    ]
```

---

## Reference-Based Evaluation

### Exact Match

```python
@run_evaluator
def exact_match(run, example) -> EvaluationResult:
    """Check for exact string match with reference."""
    prediction = run.outputs.get("output", "").strip().lower()
    reference = example.outputs.get("answer", "").strip().lower()
    return EvaluationResult(
        key="exact_match",
        score=1.0 if prediction == reference else 0.0,
    )
```

### Semantic Similarity

```python
from langchain_openai import OpenAIEmbeddings
import numpy as np

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

@run_evaluator
def semantic_similarity(run, example) -> EvaluationResult:
    """Measure semantic similarity between prediction and reference."""
    prediction = run.outputs.get("output", "")
    reference = example.outputs.get("answer", "")

    pred_vec = embeddings.embed_query(prediction)
    ref_vec = embeddings.embed_query(reference)

    similarity = np.dot(pred_vec, ref_vec) / (
        np.linalg.norm(pred_vec) * np.linalg.norm(ref_vec)
    )

    return EvaluationResult(
        key="semantic_similarity",
        score=float(similarity),
        comment=f"Cosine similarity: {similarity:.3f}",
    )
```

### ROUGE Score

```python
from rouge_score import rouge_scorer

scorer = rouge_scorer.RougeScorer(["rouge1", "rouge2", "rougeL"], use_stemmer=True)

@run_evaluator
def rouge_evaluation(run, example) -> list[EvaluationResult]:
    """Compute ROUGE scores between prediction and reference."""
    prediction = run.outputs.get("output", "")
    reference = example.outputs.get("answer", "")

    scores = scorer.score(reference, prediction)

    return [
        EvaluationResult(key="rouge1", score=scores["rouge1"].fmeasure),
        EvaluationResult(key="rouge2", score=scores["rouge2"].fmeasure),
        EvaluationResult(key="rougeL", score=scores["rougeL"].fmeasure),
    ]
```

---

## Human Feedback

### Collecting Annotations in LangSmith

```python
# After running evaluation, annotate results in LangSmith UI
# Or programmatically via API:

from langsmith import Client

client = Client()

# Add feedback to a specific run
client.create_feedback(
    run_id="run-id-here",
    key="human_quality",
    score=0.8,
    comment="Good answer but missing one key detail about LCEL streaming.",
)
```

### Human-in-the-Loop Evaluation Pipeline

```python
@run_evaluator
def flag_for_human_review(run, example) -> EvaluationResult:
    """Flag low-confidence outputs for human review."""
    prediction = run.outputs.get("output", "")

    # Auto-check basic quality
    issues = []
    if len(prediction.split()) < 5:
        issues.append("very_short")
    if "I don't know" in prediction or "I'm not sure" in prediction:
        issues.append("uncertain")
    if not prediction.strip():
        issues.append("empty")

    needs_review = len(issues) > 0
    return EvaluationResult(
        key="needs_human_review",
        score=0.0 if needs_review else 1.0,
        comment=f"Issues: {', '.join(issues)}" if issues else "Looks good",
    )
```

---

## Dataset Regression Testing

### Automated Regression Pipeline

```python
from datetime import datetime

def run_regression_suite(chain_fn, dataset_name: str, min_scores: dict):
    """Run regression tests and fail if scores drop below thresholds."""
    results = evaluate(
        chain_fn,
        data=dataset_name,
        evaluators=["qa", "cot_qa", length_check, llm_judge_relevance],
        experiment_prefix=f"regression-{datetime.now().strftime('%Y%m%d-%H%M')}",
    )

    metrics = results.aggregate_metrics
    failures = []

    for metric_name, min_score in min_scores.items():
        actual = metrics.get(metric_name, {}).get("mean", 0)
        if actual < min_score:
            failures.append(
                f"{metric_name}: {actual:.3f} < {min_score:.3f} (minimum)"
            )

    if failures:
        raise AssertionError(
            f"Regression test failures:\n" + "\n".join(failures)
        )

    return results

# Usage in CI
run_regression_suite(
    chain_fn=lambda inputs: {"output": rag_chain.invoke(inputs["question"])},
    dataset_name="rag-qa-golden",
    min_scores={
        "qa": 0.80,
        "relevance": 0.75,
        "length_check": 0.90,
    },
)
```

### Comparing Experiments

```python
def compare_experiments(baseline_prefix: str, candidate_prefix: str):
    """Compare two experiment runs to detect regressions."""
    projects = client.list_projects()

    baseline = None
    candidate = None
    for project in projects:
        if project.name.startswith(baseline_prefix):
            baseline = project
        if project.name.startswith(candidate_prefix):
            candidate = project

    if not baseline or not candidate:
        raise ValueError("Could not find both experiments")

    # Compare via LangSmith UI or API
    print(f"Baseline: {baseline.name}")
    print(f"Candidate: {candidate.name}")
    print("Compare at: https://smith.langchain.com/")
```

---

## A/B Testing Chains

### Running Parallel Evaluations

```python
def ab_test(chain_a, chain_b, dataset_name: str, evaluators: list):
    """Run the same evaluation on two chains and compare results."""
    results_a = evaluate(
        lambda inputs: {"output": chain_a.invoke(inputs["question"])},
        data=dataset_name,
        evaluators=evaluators,
        experiment_prefix="variant-A",
    )

    results_b = evaluate(
        lambda inputs: {"output": chain_b.invoke(inputs["question"])},
        data=dataset_name,
        evaluators=evaluators,
        experiment_prefix="variant-B",
    )

    metrics_a = results_a.aggregate_metrics
    metrics_b = results_b.aggregate_metrics

    print("A/B Test Results:")
    print(f"{'Metric':<25} {'Variant A':<15} {'Variant B':<15} {'Winner'}")
    print("-" * 70)

    for metric in metrics_a:
        score_a = metrics_a[metric].get("mean", 0)
        score_b = metrics_b.get(metric, {}).get("mean", 0)
        winner = "A" if score_a > score_b else "B" if score_b > score_a else "Tie"
        print(f"{metric:<25} {score_a:<15.3f} {score_b:<15.3f} {winner}")

    return metrics_a, metrics_b
```

### Statistical Significance

```python
from scipy import stats

def is_significant(scores_a: list[float], scores_b: list[float], alpha: float = 0.05) -> bool:
    """Determine if the difference between two sets of scores is statistically significant."""
    t_stat, p_value = stats.ttest_ind(scores_a, scores_b)
    return p_value < alpha
```

---

## Quality Metrics

### Faithfulness (Groundedness)

Measures whether the answer is supported by the provided context:

```python
@run_evaluator
def faithfulness(run, example) -> EvaluationResult:
    """Evaluate if the answer is grounded in the retrieved context."""
    prediction = run.outputs.get("output", "")
    context = run.outputs.get("context", example.inputs.get("context", ""))

    judge = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    result = judge.with_structured_output(FaithfulnessScore).invoke(
        f"Is every claim in the answer supported by the context?\n\n"
        f"Context: {context}\n\nAnswer: {prediction}"
    )
    return EvaluationResult(key="faithfulness", score=result.score, comment=result.reasoning)
```

### Answer Relevance

Measures whether the answer addresses the question:

```python
@run_evaluator
def answer_relevance(run, example) -> EvaluationResult:
    """Evaluate if the answer is relevant to the question asked."""
    question = example.inputs.get("question", "")
    prediction = run.outputs.get("output", "")

    judge = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    result = judge.with_structured_output(RelevanceScore).invoke(
        f"Does this answer address the question?\n\n"
        f"Question: {question}\n\nAnswer: {prediction}"
    )
    return EvaluationResult(key="answer_relevance", score=result.score, comment=result.reasoning)
```

### Context Relevance

Measures whether the retrieved context is relevant to the question:

```python
@run_evaluator
def context_relevance(run, example) -> EvaluationResult:
    """Evaluate if the retrieved context is relevant to the question."""
    question = example.inputs.get("question", "")
    context = run.outputs.get("context", "")

    judge = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    result = judge.with_structured_output(RelevanceScore).invoke(
        f"Is this context relevant to answering the question?\n\n"
        f"Question: {question}\n\nContext: {context}"
    )
    return EvaluationResult(key="context_relevance", score=result.score, comment=result.reasoning)
```

### Coherence

Measures whether the answer is well-structured and logically consistent:

```python
@run_evaluator
def coherence(run, example) -> EvaluationResult:
    """Evaluate the coherence and clarity of the answer."""
    prediction = run.outputs.get("output", "")

    judge = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    result = judge.with_structured_output(CoherenceScore).invoke(
        f"Rate the coherence, clarity, and logical structure of this text.\n\n"
        f"Text: {prediction}"
    )
    return EvaluationResult(key="coherence", score=result.score, comment=result.reasoning)
```

### Combined RAG Metrics

```python
def evaluate_rag_pipeline(chain_fn, dataset_name: str):
    """Run all RAG-specific metrics."""
    return evaluate(
        chain_fn,
        data=dataset_name,
        evaluators=[
            faithfulness,
            answer_relevance,
            context_relevance,
            coherence,
            length_check,
            "qa",
        ],
        experiment_prefix=f"rag-eval-{datetime.now().strftime('%Y%m%d')}",
    )
```

---

## Cost Tracking

### Per-Evaluation Cost Tracking

```python
from langchain_community.callbacks import get_openai_callback

@run_evaluator
def cost_tracker(run, example) -> EvaluationResult:
    """Track the cost of each evaluation run."""
    # Cost is tracked via LangSmith traces
    # Access token counts from the run metadata
    total_tokens = 0
    if run.extra and "metrics" in run.extra:
        total_tokens = run.extra["metrics"].get("total_tokens", 0)

    # Estimate cost (adjust rates per model)
    estimated_cost = total_tokens * 0.00002  # rough estimate

    return EvaluationResult(
        key="estimated_cost",
        score=estimated_cost,
        comment=f"Tokens: {total_tokens}, Estimated cost: ${estimated_cost:.4f}",
    )
```

### Evaluation Budget Tracking

```python
class EvaluationBudget:
    """Track cumulative evaluation costs across all runs."""

    def __init__(self, max_budget: float = 10.0):
        self.max_budget = max_budget
        self.total_spent = 0.0
        self.run_costs = []

    def track(self, cost: float, run_id: str):
        self.total_spent += cost
        self.run_costs.append({"run_id": run_id, "cost": cost})

        if self.total_spent > self.max_budget:
            raise RuntimeError(
                f"Evaluation budget exceeded: ${self.total_spent:.2f} > ${self.max_budget:.2f}"
            )

    def summary(self):
        return {
            "total_spent": self.total_spent,
            "remaining_budget": self.max_budget - self.total_spent,
            "num_runs": len(self.run_costs),
            "avg_cost_per_run": self.total_spent / max(len(self.run_costs), 1),
        }
```

### Model Cost Comparison

```python
def compare_model_costs(dataset_name: str, models: list[str]):
    """Compare evaluation scores and costs across different models."""
    results = {}

    for model_name in models:
        llm = ChatOpenAI(model=model_name, temperature=0)
        chain = build_chain(llm)

        with get_openai_callback() as cb:
            eval_result = evaluate(
                lambda inputs: {"output": chain.invoke(inputs["question"])},
                data=dataset_name,
                evaluators=["qa"],
                experiment_prefix=f"cost-compare-{model_name}",
            )

            results[model_name] = {
                "qa_score": eval_result.aggregate_metrics["qa"]["mean"],
                "total_cost": cb.total_cost,
                "total_tokens": cb.total_tokens,
                "cost_per_query": cb.total_cost / max(cb.successful_requests, 1),
            }

    print(f"{'Model':<20} {'QA Score':<12} {'Total Cost':<15} {'Cost/Query':<12}")
    print("-" * 60)
    for model, data in results.items():
        print(f"{model:<20} {data['qa_score']:<12.3f} ${data['total_cost']:<14.4f} ${data['cost_per_query']:<11.4f}")

    return results
```

---

## Evaluation Pipeline Integration

### CI/CD Integration

```python
# tests/evaluation/test_regression.py
import pytest

@pytest.mark.evaluation
def test_rag_quality_regression():
    """Fail CI if RAG quality drops below thresholds."""
    results = evaluate_rag_pipeline(
        chain_fn=lambda inputs: {
            "output": production_chain.invoke(inputs["question"]),
            "context": production_chain.retrieve(inputs["question"]),
        },
        dataset_name="rag-qa-golden",
    )

    metrics = results.aggregate_metrics

    assert metrics["faithfulness"]["mean"] >= 0.80, "Faithfulness regression"
    assert metrics["answer_relevance"]["mean"] >= 0.75, "Relevance regression"
    assert metrics["coherence"]["mean"] >= 0.70, "Coherence regression"
    assert metrics["qa"]["mean"] >= 0.80, "QA correctness regression"
```

### Scheduled Evaluation

```yaml
# .github/workflows/evaluation.yml
name: LangChain Evaluation
on:
  schedule:
    - cron: '0 6 * * 1'  # Weekly on Monday at 6 AM
  workflow_dispatch:

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r requirements-eval.txt
      - run: pytest tests/evaluation/ -m evaluation --timeout=600
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          LANGCHAIN_API_KEY: ${{ secrets.LANGCHAIN_API_KEY }}
          LANGCHAIN_TRACING_V2: "true"
```
