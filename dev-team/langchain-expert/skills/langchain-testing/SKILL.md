---
name: langchain-testing
triggers:
  - "test LangChain"
  - "mock LLM"
  - "evaluate chain"
  - "LangSmith evaluation"
  - "test RAG"
description: |
  This skill should be used when the task involves testing LangChain applications, including unit testing chains and agents, mocking LLM responses, testing tools and retrievers, setting up evaluation frameworks with LangSmith, building regression test suites, or measuring RAG pipeline quality metrics like faithfulness, relevance, and coherence.
---

# LangChain Testing and Evaluation

## Testing Strategy

LangChain applications require a multi-layered testing strategy that accounts for non-deterministic LLM outputs, external API dependencies, and complex chain interactions.

### Testing Pyramid for LangChain

```
         /  E2E Tests  \        Full pipeline with real LLMs (expensive, slow)
        / Integration    \      Chain composition with mocked LLMs
       /  Component Tests  \    Individual tools, parsers, prompts
      /   Unit Tests         \  Pure functions, data transformations
```

The foundation consists of deterministic unit tests for pure functions. Component tests verify individual LangChain components in isolation. Integration tests validate chain composition with mocked or cheap models. End-to-end tests run sparingly with real LLMs.

## Unit Testing Chains

### Testing with FakeLLM

To test chains without calling real LLMs, use FakeLLM and FakeChatModel:

```python
import pytest
from langchain_core.language_models.fake_chat_models import FakeChatModel
from langchain_core.messages import AIMessage
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

def make_fake_llm(responses: list[str]) -> FakeChatModel:
    """Create a FakeChatModel that returns predetermined responses."""
    return FakeChatModel(
        responses=[AIMessage(content=r) for r in responses]
    )

def test_summary_chain():
    fake_llm = make_fake_llm(["This is a concise summary of the document."])

    prompt = ChatPromptTemplate.from_template("Summarize: {text}")
    chain = prompt | fake_llm | StrOutputParser()

    result = chain.invoke({"text": "Long document text..."})
    assert result == "This is a concise summary of the document."

def test_chain_with_multiple_steps():
    fake_llm = make_fake_llm([
        "Category: technical",
        "This is a technical explanation of the concept.",
    ])

    classify_chain = classify_prompt | fake_llm | StrOutputParser()
    generate_chain = generate_prompt | fake_llm | StrOutputParser()

    # Test classification step
    category = classify_chain.invoke({"input": "How does TCP work?"})
    assert "technical" in category.lower()
```

### Testing Output Parsers

To verify that output parsers handle both valid and malformed LLM output:

```python
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

class Sentiment(BaseModel):
    label: str = Field(description="positive, negative, or neutral")
    confidence: float = Field(ge=0, le=1)

parser = PydanticOutputParser(pydantic_object=Sentiment)

def test_parser_valid_json():
    valid_output = '{"label": "positive", "confidence": 0.95}'
    result = parser.parse(valid_output)
    assert result.label == "positive"
    assert result.confidence == 0.95

def test_parser_with_markdown_wrapper():
    wrapped = '```json\n{"label": "negative", "confidence": 0.8}\n```'
    result = parser.parse(wrapped)
    assert result.label == "negative"

def test_parser_rejects_invalid():
    with pytest.raises(Exception):
        parser.parse("This is not JSON at all")
```

### Testing Prompt Templates

To validate that prompts render correctly with all variable combinations:

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

def test_prompt_renders_all_variables():
    prompt = ChatPromptTemplate.from_messages([
        ("system", "You are a {role} assistant."),
        ("human", "{question}"),
    ])

    messages = prompt.format_messages(role="research", question="What is LCEL?")
    assert len(messages) == 2
    assert "research" in messages[0].content
    assert "LCEL" in messages[1].content

def test_prompt_with_optional_history():
    prompt = ChatPromptTemplate.from_messages([
        ("system", "You are helpful."),
        MessagesPlaceholder("history", optional=True),
        ("human", "{input}"),
    ])

    # Without history
    messages = prompt.format_messages(input="Hello")
    assert len(messages) == 2

    # With history
    from langchain_core.messages import HumanMessage, AIMessage
    messages = prompt.format_messages(
        input="Follow up",
        history=[HumanMessage(content="Hi"), AIMessage(content="Hello!")],
    )
    assert len(messages) == 4
```

## Mocking LLMs

### FakeChatModel Patterns

To create fake models that simulate different behaviors:

```python
from langchain_core.language_models.fake_chat_models import (
    FakeChatModel,
    FakeListChatModel,
)
from langchain_core.messages import AIMessage

# Sequential responses
sequential_llm = FakeListChatModel(
    responses=["First response", "Second response", "Third response"]
)

# Cycle through responses
cycling_llm = FakeListChatModel(
    responses=["Response A", "Response B"],
    sleep=0,  # no delay
)

# Structured output simulation
def fake_structured_response():
    return FakeChatModel(
        responses=[AIMessage(content='{"name": "test", "score": 0.9}')]
    )
```

### Mocking with pytest

To mock LLM calls at the test level:

```python
from unittest.mock import AsyncMock, patch, MagicMock
from langchain_core.messages import AIMessage

@patch("myapp.chains.ChatOpenAI")
def test_chain_with_mock(mock_openai_class):
    mock_llm = MagicMock()
    mock_llm.invoke.return_value = AIMessage(content="Mocked response")
    mock_openai_class.return_value = mock_llm

    from myapp.chains import build_chain
    chain = build_chain()
    result = chain.invoke({"input": "test"})
    assert "Mocked" in result
```

### Testing Streaming

To verify streaming behavior:

```python
from langchain_core.language_models.fake_chat_models import FakeListChatModel

def test_streaming_output():
    fake_llm = FakeListChatModel(responses=["Hello world"])

    chunks = list(fake_llm.stream("test"))
    assert len(chunks) > 0
    full_response = "".join(chunk.content for chunk in chunks)
    assert "Hello world" in full_response
```

## Testing Tools

To test tools independently from agents:

```python
from langchain_core.tools import tool

@tool
def calculate(expression: str) -> str:
    """Evaluate a math expression."""
    try:
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"Error: {e}"

def test_tool_basic():
    result = calculate.invoke({"expression": "2 + 3"})
    assert result == "5"

def test_tool_error():
    result = calculate.invoke({"expression": "invalid"})
    assert "Error" in result

def test_tool_schema():
    schema = calculate.args_schema.model_json_schema()
    assert "expression" in schema["properties"]
```

## Testing Retrievers

To test retrievers with controlled document sets:

```python
from langchain_core.documents import Document
from langchain_community.vectorstores import FAISS
from langchain_community.embeddings import FakeEmbeddings

def create_test_vectorstore():
    docs = [
        Document(page_content="Python is a programming language.", metadata={"topic": "python"}),
        Document(page_content="JavaScript runs in browsers.", metadata={"topic": "javascript"}),
        Document(page_content="Rust is memory safe.", metadata={"topic": "rust"}),
    ]
    return FAISS.from_documents(docs, FakeEmbeddings(size=384))

def test_retriever_returns_relevant_docs():
    vs = create_test_vectorstore()
    retriever = vs.as_retriever(search_kwargs={"k": 2})
    docs = retriever.invoke("What programming language is memory safe?")
    assert len(docs) == 2

def test_retriever_metadata_filtering():
    vs = create_test_vectorstore()
    docs = vs.similarity_search("programming", k=1, filter={"topic": "python"})
    assert docs[0].metadata["topic"] == "python"
```

## Evaluation Frameworks

### LangSmith Evaluation

To set up automated evaluation with LangSmith:

```python
from langsmith import Client
from langsmith.evaluation import evaluate

client = Client()

# Create a dataset
dataset = client.create_dataset("qa-eval-set")
client.create_examples(
    inputs=[
        {"question": "What is LCEL?"},
        {"question": "How do agents work?"},
    ],
    outputs=[
        {"answer": "LCEL is LangChain Expression Language for composing chains."},
        {"answer": "Agents use LLMs to decide which tools to call."},
    ],
    dataset_id=dataset.id,
)

# Run evaluation
results = evaluate(
    lambda inputs: chain.invoke(inputs),
    data="qa-eval-set",
    evaluators=["qa", "cot_qa"],
    experiment_prefix="v1-gpt4o",
)

print(f"Average score: {results.aggregate_metrics}")
```

### Custom Evaluators

To write evaluators that measure domain-specific quality:

```python
from langsmith.evaluation import EvaluationResult, run_evaluator

@run_evaluator
def check_no_hallucination(run, example) -> EvaluationResult:
    """Check that the answer does not contain information absent from context."""
    prediction = run.outputs.get("output", "")
    reference = example.outputs.get("answer", "")

    # Use LLM to judge
    judge = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    judgment = judge.invoke(
        f"Does the prediction contain claims not supported by the reference?\n\n"
        f"Reference: {reference}\nPrediction: {prediction}\n\n"
        f"Answer YES or NO, then explain."
    )

    is_faithful = "NO" in judgment.content.upper().split("\n")[0]
    return EvaluationResult(
        key="no_hallucination",
        score=1.0 if is_faithful else 0.0,
        comment=judgment.content,
    )

results = evaluate(
    chain_fn,
    data="qa-eval-set",
    evaluators=[check_no_hallucination],
)
```

## Regression Testing

### Snapshot Testing for Chains

To detect unintended changes in chain output:

```python
import json
from pathlib import Path

SNAPSHOT_DIR = Path("tests/snapshots")

def test_chain_output_snapshot():
    result = chain.invoke({"question": "What is LCEL?"})

    snapshot_file = SNAPSHOT_DIR / "lcel_answer.json"
    if snapshot_file.exists():
        expected = json.loads(snapshot_file.read_text())
        # Use semantic similarity instead of exact match
        similarity = compute_similarity(result, expected["output"])
        assert similarity > 0.85, f"Output diverged: similarity={similarity}"
    else:
        snapshot_file.parent.mkdir(parents=True, exist_ok=True)
        snapshot_file.write_text(json.dumps({"output": result}, indent=2))
```

### Dataset-Based Regression

To run regression tests against a golden dataset:

```python
def test_rag_regression():
    """Ensure RAG pipeline quality does not degrade."""
    results = evaluate(
        rag_chain_fn,
        data="rag-golden-set",
        evaluators=["qa", "context_relevance"],
        experiment_prefix=f"regression-{datetime.now().strftime('%Y%m%d')}",
    )

    metrics = results.aggregate_metrics
    assert metrics["qa"]["mean"] >= 0.8, f"QA score dropped: {metrics['qa']['mean']}"
    assert metrics["context_relevance"]["mean"] >= 0.7
```

## Test Organization

Organize LangChain tests by layer:

```
tests/
  unit/
    test_prompts.py          # Prompt template rendering
    test_parsers.py          # Output parser logic
    test_utils.py            # Helper functions
  component/
    test_tools.py            # Individual tool behavior
    test_retrievers.py       # Retriever configuration
    test_memory.py           # Memory backends
  integration/
    test_chains.py           # Chain composition with fake LLMs
    test_agents.py           # Agent behavior with fake LLMs
    test_rag_pipeline.py     # RAG chain with fake embeddings
  e2e/
    test_full_pipeline.py    # Real LLM calls (expensive, CI-gated)
  evaluation/
    test_eval_qa.py          # LangSmith evaluation runs
    test_eval_rag.py         # RAG quality evaluation
  conftest.py                # Shared fixtures
```

### Shared Fixtures

```python
# conftest.py
import pytest
from langchain_core.language_models.fake_chat_models import FakeListChatModel
from langchain_community.embeddings import FakeEmbeddings

@pytest.fixture
def fake_llm():
    return FakeListChatModel(responses=["Default test response"])

@pytest.fixture
def fake_embeddings():
    return FakeEmbeddings(size=384)

@pytest.fixture
def sample_documents():
    from langchain_core.documents import Document
    return [
        Document(page_content="Doc 1 content", metadata={"source": "test"}),
        Document(page_content="Doc 2 content", metadata={"source": "test"}),
    ]
```

## References

For detailed testing patterns including FakeLLM/FakeChatModel usage, mocking retrievers, testing tools, integration tests, snapshot testing, streaming tests, callback tests, and error path testing, see [testing-patterns.md](references/testing-patterns.md).

For comprehensive evaluation guidance including LangSmith evaluators, custom evaluators, reference-based evaluation, human feedback, dataset regression testing, A/B testing chains, quality metrics, and cost tracking, see [evaluation-guide.md](references/evaluation-guide.md).
