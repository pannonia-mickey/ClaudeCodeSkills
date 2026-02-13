# LangChain Testing Patterns

Comprehensive reference for testing LangChain applications at every layer, from unit tests with fake models to integration tests with mocked dependencies.

---

## FakeLLM and FakeChatModel

### FakeListChatModel for Sequential Responses

```python
from langchain_core.language_models.fake_chat_models import FakeListChatModel
from langchain_core.messages import AIMessage

# Returns responses in order, cycling when exhausted
fake_llm = FakeListChatModel(
    responses=["Response 1", "Response 2", "Response 3"],
    sleep=0,
)

# First call returns "Response 1", second returns "Response 2", etc.
result1 = fake_llm.invoke("Any input")
result2 = fake_llm.invoke("Different input")
assert result1.content == "Response 1"
assert result2.content == "Response 2"
```

### FakeChatModel for Controlled Behavior

```python
from langchain_core.language_models.fake_chat_models import FakeChatModel

# Single response
fake = FakeChatModel(responses=[AIMessage(content="Fixed response")])

# With tool calls
from langchain_core.messages import AIMessage

tool_call_response = AIMessage(
    content="",
    tool_calls=[{
        "id": "call_1",
        "name": "search",
        "args": {"query": "test"},
    }],
)

fake_with_tools = FakeChatModel(responses=[tool_call_response])
```

### FakeEmbeddings

```python
from langchain_community.embeddings import FakeEmbeddings

# Deterministic fake embeddings for testing
fake_embeddings = FakeEmbeddings(size=384)

# Generate embeddings without API calls
vectors = fake_embeddings.embed_documents(["doc 1", "doc 2"])
assert len(vectors) == 2
assert len(vectors[0]) == 384
```

### GenericFakeChatModel for Token Streaming

```python
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessageChunk

# Simulates token-by-token streaming
fake_streaming = GenericFakeChatModel(
    messages=iter([
        AIMessageChunk(content="Hello"),
        AIMessageChunk(content=" world"),
        AIMessageChunk(content="!"),
    ])
)
```

---

## Mocking Retrievers

### Using a Fake Retriever

```python
from langchain_core.retrievers import BaseRetriever
from langchain_core.documents import Document
from typing import List

class FakeRetriever(BaseRetriever):
    """Retriever that returns predetermined documents."""
    documents: list[Document]

    def _get_relevant_documents(self, query: str) -> list[Document]:
        return self.documents

fake_retriever = FakeRetriever(documents=[
    Document(page_content="LCEL uses the pipe operator.", metadata={"source": "docs"}),
    Document(page_content="Chains are composed of Runnables.", metadata={"source": "docs"}),
])

def test_rag_chain_uses_context():
    fake_llm = FakeListChatModel(responses=["LCEL uses pipes for composition."])
    chain = build_rag_chain(fake_retriever, fake_llm)
    result = chain.invoke("What is LCEL?")
    assert "pipe" in result.lower() or "LCEL" in result
```

### Mocking Vector Store Retrieval

```python
from unittest.mock import MagicMock

def test_retriever_integration():
    mock_vectorstore = MagicMock()
    mock_vectorstore.as_retriever.return_value = fake_retriever
    mock_vectorstore.similarity_search.return_value = [
        Document(page_content="Mock result", metadata={"score": 0.95}),
    ]

    docs = mock_vectorstore.similarity_search("query", k=3)
    assert len(docs) == 1
    assert docs[0].metadata["score"] == 0.95
```

### Testing Retriever with Real Vector Store (In-Memory)

```python
from langchain_community.vectorstores import FAISS
from langchain_community.embeddings import FakeEmbeddings

def create_test_retriever(docs: list[Document], k: int = 3):
    """Create a real but lightweight retriever for testing."""
    vectorstore = FAISS.from_documents(docs, FakeEmbeddings(size=384))
    return vectorstore.as_retriever(search_kwargs={"k": k})

def test_retriever_returns_correct_count():
    docs = [Document(page_content=f"Document {i}") for i in range(10)]
    retriever = create_test_retriever(docs, k=3)
    results = retriever.invoke("query")
    assert len(results) == 3
```

---

## Testing Tools

### Testing @tool Functions

```python
from langchain_core.tools import tool

@tool
def multiply(a: int, b: int) -> str:
    """Multiply two numbers."""
    return str(a * b)

def test_tool_invoke():
    result = multiply.invoke({"a": 3, "b": 4})
    assert result == "12"

def test_tool_metadata():
    assert multiply.name == "multiply"
    assert "Multiply" in multiply.description

def test_tool_schema():
    schema = multiply.args_schema.model_json_schema()
    assert "a" in schema["properties"]
    assert "b" in schema["properties"]
    assert schema["properties"]["a"]["type"] == "integer"
```

### Testing Tool Error Handling

```python
@tool
def divide(a: float, b: float) -> str:
    """Divide a by b."""
    if b == 0:
        return "Error: Division by zero is not allowed."
    return str(a / b)

def test_division_by_zero():
    result = divide.invoke({"a": 10, "b": 0})
    assert "Error" in result
    assert "zero" in result.lower()

def test_normal_division():
    result = divide.invoke({"a": 10, "b": 2})
    assert result == "5.0"
```

### Testing Async Tools

```python
import pytest

@tool
async def async_fetch(url: str) -> str:
    """Fetch a URL asynchronously."""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.text()

@pytest.mark.asyncio
async def test_async_tool():
    with aiohttp.test_utils.TestServer(mock_app) as server:
        result = await async_fetch.ainvoke({"url": str(server.make_url("/"))})
        assert "expected content" in result
```

### Testing BaseTool Subclasses

```python
def test_custom_tool_initialization():
    tool = APISearchTool(api_base_url="https://test.api.com", api_key="test-key")
    assert tool.name == "api_search"
    assert tool.api_key == "test-key"

def test_custom_tool_with_mocked_api(requests_mock):
    requests_mock.get(
        "https://test.api.com/search",
        json={"results": [{"title": "Result 1", "summary": "Summary"}], "total": 1},
    )
    tool = APISearchTool(api_base_url="https://test.api.com", api_key="test-key")
    result = tool.invoke({"query": "test", "filters": {}, "page": 1})
    assert "Result 1" in result
```

---

## Integration Tests

### Testing Chain Composition

```python
def test_full_rag_chain_integration():
    """Test complete RAG chain with fake components."""
    # Setup
    docs = [
        Document(page_content="LCEL composes chains with the pipe operator."),
        Document(page_content="Agents use tools to interact with external systems."),
    ]
    retriever = create_test_retriever(docs)
    fake_llm = FakeListChatModel(responses=[
        "LCEL uses the pipe operator to compose chains together."
    ])

    # Build chain
    chain = build_rag_chain(retriever, fake_llm)

    # Test
    result = chain.invoke("How does LCEL work?")
    assert isinstance(result, str)
    assert len(result) > 0
```

### Testing Agent Behavior

```python
def test_agent_uses_correct_tool():
    """Verify the agent selects the right tool based on the query."""
    # Fake LLM that simulates tool calling
    tool_call_msg = AIMessage(
        content="",
        tool_calls=[{"id": "1", "name": "calculator", "args": {"expression": "2+2"}}],
    )
    final_msg = AIMessage(content="The answer is 4.")

    fake_llm = FakeListChatModel(responses=[
        tool_call_msg.content,
        final_msg.content,
    ])

    # Note: for tool-calling agents, you may need to mock at a deeper level
    # or use the actual agent with a FakeChatModel that returns tool_calls
```

### Testing Conversational Chains

```python
def test_conversation_maintains_history():
    """Verify that conversation history is preserved across turns."""
    fake_llm = FakeListChatModel(responses=[
        "My name is Claude.",
        "You asked me my name, and I said Claude.",
    ])

    chain = build_conversational_chain(fake_llm)

    result1 = chain.invoke(
        {"input": "What is your name?"},
        config={"configurable": {"session_id": "test-1"}},
    )
    assert "Claude" in result1

    result2 = chain.invoke(
        {"input": "What did I just ask?"},
        config={"configurable": {"session_id": "test-1"}},
    )
    assert isinstance(result2, str)
```

---

## Snapshot Testing

### Output Snapshot Testing

```python
import json
from pathlib import Path

SNAPSHOT_DIR = Path("tests/snapshots")

def snapshot_test(name: str, actual: str, threshold: float = 0.85):
    """Compare output against a stored snapshot using semantic similarity."""
    snapshot_file = SNAPSHOT_DIR / f"{name}.json"

    if not snapshot_file.exists():
        # First run: create snapshot
        snapshot_file.parent.mkdir(parents=True, exist_ok=True)
        snapshot_file.write_text(json.dumps({"output": actual}, indent=2))
        return

    expected = json.loads(snapshot_file.read_text())["output"]

    # For deterministic outputs, use exact match
    if actual == expected:
        return

    # For LLM outputs, use embedding similarity
    from langchain_community.embeddings import FakeEmbeddings
    import numpy as np

    embeddings = FakeEmbeddings(size=384)
    vec_actual = embeddings.embed_query(actual)
    vec_expected = embeddings.embed_query(expected)

    similarity = np.dot(vec_actual, vec_expected) / (
        np.linalg.norm(vec_actual) * np.linalg.norm(vec_expected)
    )
    assert similarity >= threshold, (
        f"Snapshot mismatch for '{name}': similarity={similarity:.3f} < {threshold}"
    )
```

### Prompt Snapshot Testing

```python
def test_prompt_snapshot():
    """Ensure prompt templates haven't changed unexpectedly."""
    prompt = build_rag_prompt()
    rendered = prompt.format_messages(
        context="Test context",
        question="Test question",
    )
    serialized = [{"role": m.type, "content": m.content} for m in rendered]

    snapshot_file = SNAPSHOT_DIR / "rag_prompt.json"
    if snapshot_file.exists():
        expected = json.loads(snapshot_file.read_text())
        assert serialized == expected, "Prompt template changed unexpectedly"
    else:
        snapshot_file.write_text(json.dumps(serialized, indent=2))
```

---

## Testing Streaming

### Verifying Stream Output

```python
def test_chain_streams_tokens():
    fake_llm = FakeListChatModel(responses=["Hello world"])

    chain = prompt | fake_llm | StrOutputParser()
    chunks = list(chain.stream({"input": "test"}))

    assert len(chunks) > 0
    full_output = "".join(chunks)
    assert "Hello world" in full_output

@pytest.mark.asyncio
async def test_async_streaming():
    fake_llm = FakeListChatModel(responses=["Async response"])

    chain = prompt | fake_llm | StrOutputParser()
    chunks = []
    async for chunk in chain.astream({"input": "test"}):
        chunks.append(chunk)

    assert len(chunks) > 0
```

### Testing Stream Events

```python
@pytest.mark.asyncio
async def test_stream_events():
    fake_llm = FakeListChatModel(responses=["Event test"])
    chain = prompt | fake_llm | StrOutputParser()

    events = []
    async for event in chain.astream_events({"input": "test"}, version="v2"):
        events.append(event["event"])

    assert "on_chain_start" in events
    assert "on_chain_end" in events
```

---

## Testing Callbacks

### Verifying Callback Invocation

```python
from langchain_core.callbacks import BaseCallbackHandler

class TrackingHandler(BaseCallbackHandler):
    def __init__(self):
        self.events = []

    def on_chain_start(self, serialized, inputs, **kwargs):
        self.events.append(("chain_start", serialized.get("name")))

    def on_chain_end(self, outputs, **kwargs):
        self.events.append(("chain_end", None))

    def on_llm_start(self, serialized, prompts, **kwargs):
        self.events.append(("llm_start", None))

    def on_llm_end(self, response, **kwargs):
        self.events.append(("llm_end", None))

def test_callbacks_fire_in_order():
    tracker = TrackingHandler()
    fake_llm = FakeListChatModel(responses=["test"])
    chain = prompt | fake_llm | StrOutputParser()

    chain.invoke({"input": "test"}, config={"callbacks": [tracker]})

    event_types = [e[0] for e in tracker.events]
    assert "chain_start" in event_types
    assert "llm_start" in event_types
    assert "llm_end" in event_types
    assert "chain_end" in event_types
```

### Testing Custom Callback Handlers

```python
class CostTracker(BaseCallbackHandler):
    def __init__(self):
        self.total_tokens = 0

    def on_llm_end(self, response, **kwargs):
        if hasattr(response, "llm_output") and response.llm_output:
            usage = response.llm_output.get("token_usage", {})
            self.total_tokens += usage.get("total_tokens", 0)

def test_cost_tracker():
    tracker = CostTracker()
    # Use real LLM or mock with token usage info
    # For unit test, directly test the handler
    from langchain_core.outputs import LLMResult

    mock_response = LLMResult(
        generations=[[]],
        llm_output={"token_usage": {"total_tokens": 150}},
    )
    tracker.on_llm_end(mock_response)
    assert tracker.total_tokens == 150
```

---

## Testing Error Paths

### Retry Behavior

```python
def test_retry_on_failure():
    """Verify that .with_retry() retries on transient errors."""
    call_count = 0

    class FailThenSucceedLLM(FakeChatModel):
        def _generate(self, messages, stop=None, **kwargs):
            nonlocal call_count
            call_count += 1
            if call_count < 3:
                raise ConnectionError("Transient failure")
            return super()._generate(messages, stop, **kwargs)

    llm = FailThenSucceedLLM(responses=[AIMessage(content="Success")])
    resilient = llm.with_retry(stop_after_attempt=3)

    result = resilient.invoke("test")
    assert result.content == "Success"
    assert call_count == 3
```

### Fallback Behavior

```python
def test_fallback_on_error():
    """Verify that .with_fallbacks() switches to fallback LLM."""
    class AlwaysFailLLM(FakeChatModel):
        def _generate(self, messages, stop=None, **kwargs):
            raise RuntimeError("Primary LLM is down")

    primary = AlwaysFailLLM(responses=[])
    fallback = FakeListChatModel(responses=["Fallback response"])

    resilient = primary.with_fallbacks([fallback])
    result = resilient.invoke("test")
    assert result.content == "Fallback response"
```

### Tool Error Recovery

```python
def test_agent_recovers_from_tool_error():
    """Verify agent handles tool errors gracefully."""
    @tool
    def failing_tool(input: str) -> str:
        """A tool that always fails."""
        raise Exception("Tool is broken")

    executor = AgentExecutor(
        agent=agent,
        tools=[failing_tool],
        handle_parsing_errors=True,
        max_iterations=3,
    )

    # Agent should not crash; it should handle the error
    result = executor.invoke({"input": "Use the tool"})
    assert "output" in result
```

---

## Test Configuration

### pytest Configuration

```ini
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    "unit: Unit tests (no external dependencies)",
    "integration: Integration tests (mocked dependencies)",
    "e2e: End-to-end tests (real LLM calls)",
    "evaluation: LangSmith evaluation tests",
]
testpaths = ["tests"]
asyncio_mode = "auto"
```

### Running Test Layers Separately

```bash
# Fast unit tests
pytest -m unit

# Integration tests
pytest -m integration

# Expensive E2E tests (CI only)
pytest -m e2e --timeout=120

# Evaluation tests
pytest -m evaluation --timeout=300
```

### Environment-Based Test Skipping

```python
import os
import pytest

requires_openai = pytest.mark.skipif(
    not os.environ.get("OPENAI_API_KEY"),
    reason="Requires OPENAI_API_KEY",
)

requires_langsmith = pytest.mark.skipif(
    not os.environ.get("LANGCHAIN_API_KEY"),
    reason="Requires LANGCHAIN_API_KEY for LangSmith",
)

@requires_openai
def test_with_real_llm():
    """This test only runs when OPENAI_API_KEY is set."""
    ...
```
