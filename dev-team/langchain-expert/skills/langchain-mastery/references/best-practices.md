# LangChain Best Practices

Comprehensive guide to production-grade LangChain development covering LCEL patterns, error handling, streaming, batch processing, caching, cost optimization, and structured output.

---

## LCEL Composition Patterns

### Basic Chain Composition

The pipe operator is the foundation of LCEL. Every component in a pipe must be a Runnable.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI

# Linear chain
chain = (
    ChatPromptTemplate.from_template("Summarize: {text}")
    | ChatOpenAI(model="gpt-4o-mini", temperature=0)
    | StrOutputParser()
)
```

### Parallel Execution

Use `RunnableParallel` to fan out work and merge results:

```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough

analysis = RunnableParallel(
    summary=summary_chain,
    sentiment=sentiment_chain,
    entities=entity_chain,
    original=RunnablePassthrough(),
)

# All three chains execute concurrently
result = analysis.invoke({"text": "..."})
# result = {"summary": "...", "sentiment": "...", "entities": [...], "original": {...}}
```

### Branching and Routing

Route inputs to different chains based on classification:

```python
from langchain_core.runnables import RunnableBranch, RunnableLambda

# Classify then route
classifier = classify_prompt | llm | StrOutputParser()

def route(info):
    topic = info["topic"].lower()
    if "technical" in topic:
        return technical_chain
    elif "billing" in topic:
        return billing_chain
    return general_chain

routed_chain = (
    RunnableParallel(
        topic=classifier,
        input=RunnablePassthrough(),
    )
    | RunnableLambda(route)
)
```

### Nested Chains and Passthrough

Preserve original input alongside derived values:

```python
chain = (
    RunnableParallel(
        context=retriever | format_docs,
        question=RunnablePassthrough(),
        history=RunnableLambda(get_history),
    )
    | prompt
    | llm
    | parser
)
```

### Dynamic Chain Construction

Build chains dynamically based on configuration:

```python
def build_chain(config: dict) -> Runnable:
    steps = []

    if config.get("preprocess"):
        steps.append(RunnableLambda(preprocess))

    steps.extend([prompt, llm, parser])

    if config.get("postprocess"):
        steps.append(RunnableLambda(postprocess))

    chain = steps[0]
    for step in steps[1:]:
        chain = chain | step
    return chain
```

---

## Error Handling

### Retry with Exponential Backoff

Always attach retry logic to LLM calls to handle transient API failures:

```python
from langchain_core.runnables import RunnableConfig

robust_llm = ChatOpenAI(model="gpt-4o").with_retry(
    stop_after_attempt=3,
    wait_exponential_jitter=True,
    retry_if_exception_type=(TimeoutError, ConnectionError),
)
```

### Fallback Chains

Provide fallback models for graceful degradation:

```python
primary = ChatOpenAI(model="gpt-4o", temperature=0)
fallback = ChatOpenAI(model="gpt-4o-mini", temperature=0)

resilient_llm = primary.with_fallbacks(
    [fallback],
    exceptions_to_handle=(Exception,),
)

# If gpt-4o fails, automatically retries with gpt-4o-mini
chain = prompt | resilient_llm | parser
```

### Combined Retry and Fallback

```python
production_llm = (
    ChatOpenAI(model="gpt-4o", temperature=0)
    .with_retry(stop_after_attempt=2, wait_exponential_jitter=True)
    .with_fallbacks([
        ChatOpenAI(model="gpt-4o-mini", temperature=0)
        .with_retry(stop_after_attempt=2)
    ])
)
```

### Exception Handling in Custom Runnables

```python
from langchain_core.runnables import RunnableLambda

def safe_parse(text: str) -> dict:
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        return {"raw_text": text, "parse_error": True}

safe_parser = RunnableLambda(safe_parse)
chain = prompt | llm | StrOutputParser() | safe_parser
```

### Output Parser Error Recovery

```python
from langchain.output_parsers import RetryWithErrorOutputParser, OutputFixingParser

# Automatic retry: sends the error back to the LLM to fix
retry_parser = RetryWithErrorOutputParser.from_llm(
    parser=pydantic_parser,
    llm=ChatOpenAI(model="gpt-4o-mini"),
    max_retries=2,
)

# Automatic fixing: asks LLM to fix malformed output
fixing_parser = OutputFixingParser.from_llm(
    parser=pydantic_parser,
    llm=ChatOpenAI(model="gpt-4o-mini"),
)
```

---

## Streaming

### Token-Level Streaming

```python
# Synchronous streaming
for chunk in chain.stream({"question": "Explain LCEL"}):
    print(chunk, end="", flush=True)

# Async streaming
async for chunk in chain.astream({"question": "Explain LCEL"}):
    await send_to_client(chunk)
```

### Streaming Events (Advanced)

Use `.astream_events()` for granular control over what gets streamed:

```python
async for event in chain.astream_events(
    {"question": "What is LCEL?"},
    version="v2",
):
    kind = event["event"]
    if kind == "on_chat_model_stream":
        token = event["data"]["chunk"].content
        if token:
            print(token, end="", flush=True)
    elif kind == "on_chain_end":
        print(f"\nChain finished: {event['name']}")
```

### Streaming with FastAPI

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def generate_stream(question: str):
    async for chunk in chain.astream({"question": question}):
        yield f"data: {chunk}\n\n"

@app.post("/stream")
async def stream_endpoint(request: QuestionRequest):
    return StreamingResponse(
        generate_stream(request.question),
        media_type="text/event-stream",
    )
```

---

## Batch Processing

### Basic Batching

```python
inputs = [{"question": q} for q in questions]

# Process with controlled concurrency
results = chain.batch(inputs, config={"max_concurrency": 5})

# Async batch
results = await chain.abatch(inputs, config={"max_concurrency": 10})
```

### Batch with Error Handling

```python
results = chain.batch(
    inputs,
    config={"max_concurrency": 5},
    return_exceptions=True,
)

successes = [r for r in results if not isinstance(r, Exception)]
failures = [r for r in results if isinstance(r, Exception)]
```

### Rate-Limited Batching

```python
import asyncio
from itertools import islice

async def rate_limited_batch(chain, inputs, batch_size=10, delay=1.0):
    results = []
    it = iter(inputs)
    while batch := list(islice(it, batch_size)):
        batch_results = await chain.abatch(batch, config={"max_concurrency": batch_size})
        results.extend(batch_results)
        await asyncio.sleep(delay)
    return results
```

---

## Caching

### SQLite Cache (Development)

```python
from langchain_core.globals import set_llm_cache
from langchain_community.cache import SQLiteCache

set_llm_cache(SQLiteCache(database_path=".langchain_cache.db"))
```

### Redis Cache (Production)

```python
from langchain_community.cache import RedisCache
from redis import Redis

set_llm_cache(RedisCache(redis_=Redis(host="localhost", port=6379, db=0)))
```

### Semantic Cache

Cache based on semantic similarity rather than exact input matching:

```python
from langchain_community.cache import RedisSemanticCache
from langchain_openai import OpenAIEmbeddings

set_llm_cache(RedisSemanticCache(
    redis_url="redis://localhost:6379",
    embedding=OpenAIEmbeddings(),
    score_threshold=0.95,
))
```

### Per-Chain Caching

Disable caching for chains where freshness matters:

```python
# This LLM call will not use the global cache
uncached_llm = ChatOpenAI(model="gpt-4o", cache=False)
```

---

## Cost Optimization

### Model Tiering

Use cheaper models for simple tasks and reserve expensive models for complex reasoning:

```python
cheap_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
expensive_llm = ChatOpenAI(model="gpt-4o", temperature=0)

# Classify with cheap model, reason with expensive model
classifier = classify_prompt | cheap_llm | StrOutputParser()
reasoner = reasoning_prompt | expensive_llm | StrOutputParser()

chain = (
    RunnableParallel(
        classification=classifier,
        input=RunnablePassthrough(),
    )
    | RunnableLambda(route_by_complexity)
)
```

### Token Usage Tracking

```python
from langchain_community.callbacks import get_openai_callback

with get_openai_callback() as cb:
    result = chain.invoke({"question": "Complex question"})
    print(f"Prompt tokens: {cb.prompt_tokens}")
    print(f"Completion tokens: {cb.completion_tokens}")
    print(f"Total cost: ${cb.total_cost:.4f}")
```

### Prompt Optimization

Reduce token count without sacrificing quality:

```python
# Use system messages efficiently
prompt = ChatPromptTemplate.from_messages([
    ("system", "Expert assistant. Be concise."),  # Short system prompt
    ("human", "{question}"),
])

# Limit context window for RAG
def format_docs_with_limit(docs, max_tokens=2000):
    formatted = []
    token_count = 0
    for doc in docs:
        doc_tokens = len(doc.page_content.split())  # approximate
        if token_count + doc_tokens > max_tokens:
            break
        formatted.append(doc.page_content)
        token_count += doc_tokens
    return "\n\n".join(formatted)
```

### Embedding Cost Reduction

```python
# Cache embeddings to avoid re-computing
from langchain_community.embeddings import CacheBackedEmbeddings
from langchain_community.storage import LocalFileStore

store = LocalFileStore("./embedding_cache")
cached_embeddings = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings=OpenAIEmbeddings(),
    document_embedding_cache=store,
    namespace="openai-embeddings",
)
```

---

## Structured Output

### Using `.with_structured_output()` (Preferred)

```python
from pydantic import BaseModel, Field

class AnalysisResult(BaseModel):
    summary: str = Field(description="Brief summary of the analysis")
    confidence: float = Field(ge=0, le=1, description="Confidence score")
    key_points: list[str] = Field(description="Main points extracted")
    category: str = Field(description="Category classification")

structured_llm = ChatOpenAI(model="gpt-4o").with_structured_output(AnalysisResult)

chain = prompt | structured_llm
result: AnalysisResult = chain.invoke({"text": "..."})
```

### Enum-Based Classification

```python
from enum import Enum

class Sentiment(str, Enum):
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"

class SentimentResult(BaseModel):
    sentiment: Sentiment
    confidence: float = Field(ge=0, le=1)
    reasoning: str

structured_llm = ChatOpenAI(model="gpt-4o-mini").with_structured_output(SentimentResult)
```

### JSON Mode Fallback

For models that do not support `.with_structured_output()`:

```python
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser(pydantic_object=AnalysisResult)

chain = (
    prompt.partial(format_instructions=parser.get_format_instructions())
    | llm.bind(response_format={"type": "json_object"})
    | parser
)
```

### Streaming Structured Output

```python
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser(pydantic_object=AnalysisResult)

# Stream partial JSON as it's generated
async for partial in (prompt | llm | parser).astream({"text": "..."}):
    print(partial)  # Partial dict that grows as tokens arrive
```

---

## Dependency Management

### Pinning LangChain Versions

Always pin versions in production to avoid breaking changes:

```
langchain-core==0.3.x
langchain==0.3.x
langchain-openai==0.2.x
langchain-community==0.3.x
```

### Import Best Practices

Prefer importing from `langchain-core` and provider-specific packages over `langchain-community`:

```python
# Good: explicit, stable imports
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI, OpenAIEmbeddings

# Avoid: community imports when core/provider packages exist
# from langchain.chat_models import ChatOpenAI  # deprecated path
```

---

## Production Checklist

1. **Retry and fallback** on every LLM call
2. **Caching** enabled for deterministic prompts
3. **Streaming** wired through to the client
4. **Cost tracking** via callbacks or LangSmith
5. **Structured output** with Pydantic models for all machine-consumed responses
6. **Rate limiting** for batch operations
7. **Timeout configuration** on all external calls
8. **Error handling** with graceful degradation
9. **Version pinning** for all LangChain packages
10. **LangSmith tracing** enabled in all environments
