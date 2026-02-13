---
name: langchain-mastery
triggers:
  - "LangChain chain"
  - "LCEL"
  - "LangChain prompt"
  - "LangChain output parser"
  - "LangSmith"
description: |
  This skill should be used when the task involves core LangChain development including composing chains with LCEL, designing prompt templates, implementing output parsers, configuring callbacks, or integrating LangSmith for tracing and evaluation. It covers the foundational building blocks that all LangChain applications depend on.
---

# LangChain Mastery: LCEL, Prompts, Parsers, Callbacks, and LangSmith

## LCEL Fundamentals

LCEL (LangChain Expression Language) is the declarative composition layer for building LangChain pipelines. Treat every component as a Runnable and compose them with the pipe operator.

### Core Runnable Primitives

To create a simple chain, pipe a prompt into a model into a parser:

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant specialized in {domain}."),
    ("human", "{question}")
])

chain = prompt | ChatOpenAI(model="gpt-4o") | StrOutputParser()

result = chain.invoke({"domain": "physics", "question": "Explain entropy"})
```

To run multiple branches in parallel, use RunnableParallel:

```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough

parallel_chain = RunnableParallel(
    summary=summary_chain,
    keywords=keyword_chain,
    sentiment=sentiment_chain,
)

results = parallel_chain.invoke({"text": "Some input text"})
```

To inject custom logic, use RunnableLambda:

```python
from langchain_core.runnables import RunnableLambda

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

chain = retriever | RunnableLambda(format_docs) | prompt | llm | parser
```

To implement conditional routing, use RunnableBranch:

```python
from langchain_core.runnables import RunnableBranch

branch = RunnableBranch(
    (lambda x: "code" in x["topic"], code_chain),
    (lambda x: "math" in x["topic"], math_chain),
    general_chain,  # default
)
```

### Streaming and Async

To enable streaming from the start, call `.stream()` instead of `.invoke()`:

```python
for chunk in chain.stream({"question": "Explain quantum computing"}):
    print(chunk, end="", flush=True)
```

To use async execution for concurrent workloads:

```python
import asyncio

async def run_concurrent():
    results = await asyncio.gather(
        chain.ainvoke({"question": "Topic A"}),
        chain.ainvoke({"question": "Topic B"}),
    )
    return results
```

To process multiple inputs efficiently, use batch:

```python
results = chain.batch(
    [{"question": "Q1"}, {"question": "Q2"}, {"question": "Q3"}],
    config={"max_concurrency": 3}
)
```

### Configurable Runnables

To make chains configurable at runtime without rebuilding them, use `.configurable_fields()` and `.configurable_alternatives()`:

```python
from langchain_core.runnables import ConfigurableField

configurable_llm = ChatOpenAI(model="gpt-4o-mini").configurable_fields(
    model_name=ConfigurableField(id="model", name="Model Name")
)

chain = prompt | configurable_llm | parser

# Override at invocation time
result = chain.invoke(
    {"question": "Complex reasoning task"},
    config={"configurable": {"model": "gpt-4o"}}
)
```

## Prompt Templates

### ChatPromptTemplate Patterns

To build structured multi-turn prompts:

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a {role}. Respond in {language}."),
    MessagesPlaceholder("chat_history", optional=True),
    ("human", "{input}"),
])
```

To create few-shot prompts with dynamic example selection:

```python
from langchain_core.prompts import FewShotChatMessagePromptTemplate

example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}"),
])

few_shot = FewShotChatMessagePromptTemplate(
    example_selector=semantic_selector,
    example_prompt=example_prompt,
)

full_prompt = ChatPromptTemplate.from_messages([
    ("system", "Classify the sentiment of the following text."),
    few_shot,
    ("human", "{input}"),
])
```

### Prompt Composition

To compose partial prompts for reuse across chains:

```python
system_base = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
])

with_history = system_base + MessagesPlaceholder("history") + ("human", "{input}")
without_history = system_base + ("human", "{input}")
```

## Output Parsers

### Structured Output with Pydantic

To parse LLM output into typed Pydantic models:

```python
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

class MovieReview(BaseModel):
    title: str = Field(description="The movie title")
    rating: float = Field(description="Rating from 0 to 10")
    summary: str = Field(description="Brief review summary")

parser = PydanticOutputParser(pydantic_object=MovieReview)

prompt = ChatPromptTemplate.from_messages([
    ("system", "Extract a structured movie review.\n{format_instructions}"),
    ("human", "{review_text}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | llm | parser
review: MovieReview = chain.invoke({"review_text": "..."})
```

### Using `.with_structured_output()`

To get structured output directly from compatible models (preferred approach):

```python
class SearchQuery(BaseModel):
    query: str = Field(description="The search query")
    filters: list[str] = Field(description="Metadata filters to apply")

structured_llm = ChatOpenAI(model="gpt-4o").with_structured_output(SearchQuery)
result: SearchQuery = structured_llm.invoke("Find recent papers on transformers")
```

### Resilient Parsing

To handle parsing failures gracefully, wrap parsers with retry logic:

```python
from langchain.output_parsers import RetryWithErrorOutputParser

retry_parser = RetryWithErrorOutputParser.from_llm(
    parser=parser,
    llm=ChatOpenAI(model="gpt-4o-mini"),
    max_retries=2,
)
```

## Callbacks

### Built-in and Custom Handlers

To add logging and cost tracking via callbacks:

```python
from langchain_core.callbacks import BaseCallbackHandler
from langchain_community.callbacks import get_openai_callback

class LoggingHandler(BaseCallbackHandler):
    def on_chain_start(self, serialized, inputs, **kwargs):
        print(f"Chain started: {serialized.get('name', 'unknown')}")

    def on_chain_end(self, outputs, **kwargs):
        print(f"Chain finished with: {outputs}")

    def on_llm_error(self, error, **kwargs):
        print(f"LLM error: {error}")

with get_openai_callback() as cb:
    result = chain.invoke(input, config={"callbacks": [LoggingHandler()]})
    print(f"Tokens used: {cb.total_tokens}, Cost: ${cb.total_cost:.4f}")
```

### Streaming Callbacks

To implement real-time streaming with callbacks:

```python
from langchain_core.callbacks import AsyncCallbackHandler

class StreamHandler(AsyncCallbackHandler):
    async def on_llm_new_token(self, token: str, **kwargs):
        await websocket.send(token)
```

## LangSmith Integration

### Tracing Setup

To enable LangSmith tracing, set environment variables and annotate runs:

```python
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls__..."
os.environ["LANGCHAIN_PROJECT"] = "my-project"
```

To add metadata and tags to specific runs for filtering in the LangSmith UI:

```python
result = chain.invoke(
    {"question": "What is LCEL?"},
    config={
        "metadata": {"user_id": "u123", "session": "abc"},
        "tags": ["production", "v2"],
        "run_name": "classification_chain",
    }
)
```

### Custom Run Annotations

To wrap specific functions with tracing using the `@traceable` decorator:

```python
from langsmith import traceable

@traceable(name="preprocess_input", tags=["preprocessing"])
def preprocess(raw_input: str) -> dict:
    cleaned = raw_input.strip().lower()
    return {"input": cleaned, "length": len(cleaned)}
```

### LangSmith Evaluation

To run automated evaluations against a dataset:

```python
from langsmith import Client
from langsmith.evaluation import evaluate

client = Client()

results = evaluate(
    chain.invoke,
    data="my-eval-dataset",
    evaluators=["qa", "context_relevance"],
    experiment_prefix="v2-gpt4o",
)
```

## Configuration Best Practices

To structure a production LangChain application, follow these guidelines:

1. **Separate concerns** -- keep prompts, chains, tools, and configuration in distinct modules.
2. **Use environment-based config** -- load model names, temperatures, and API keys from environment variables or config files, never hardcode them.
3. **Version prompts** -- store prompts in LangSmith Hub or version-controlled files, not inline strings scattered through code.
4. **Enable retry and fallback** -- always attach `.with_retry(stop_after_attempt=3)` and `.with_fallbacks([fallback_llm])` to LLM calls in production.
5. **Cache aggressively** -- use `SQLiteCache` or `RedisCache` for deterministic prompts to reduce cost and latency.

```python
from langchain_core.globals import set_llm_cache
from langchain_community.cache import SQLiteCache

set_llm_cache(SQLiteCache(database_path=".langchain_cache.db"))

production_llm = (
    ChatOpenAI(model="gpt-4o", temperature=0)
    .with_retry(stop_after_attempt=3)
    .with_fallbacks([ChatOpenAI(model="gpt-4o-mini", temperature=0)])
)
```

## References

For SOLID design principles applied to LangChain components, see [solid-patterns.md](references/solid-patterns.md).

For detailed LCEL patterns, error handling, streaming, batch processing, caching, cost optimization, and structured output best practices, see [best-practices.md](references/best-practices.md).
