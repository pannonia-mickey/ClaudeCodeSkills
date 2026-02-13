# SOLID Principles for LangChain Development

This reference provides a comprehensive mapping of SOLID design principles to LangChain's component model. Each principle is illustrated with concrete Python code examples demonstrating idiomatic LangChain patterns.

---

## Single Responsibility Principle (SRP)

### Single-Purpose Chains

Every chain should do one thing well. A chain that retrieves documents, processes them, generates a response, and sends an email has too many responsibilities.

**Violation -- a monolithic chain:**

```python
# Bad: one chain doing retrieval, formatting, generation, and side effects
def do_everything(query: str):
    docs = retriever.invoke(query)
    formatted = "\n".join(d.page_content for d in docs)
    prompt_val = prompt.format(context=formatted, question=query)
    response = llm.invoke(prompt_val)
    send_email(response)
    log_to_database(response)
    return response
```

**Correct -- decomposed single-purpose chains:**

```python
from langchain_core.runnables import RunnableLambda, RunnableParallel

# Each chain has one job
retrieval_chain = retriever | RunnableLambda(format_docs)
generation_chain = prompt | llm | StrOutputParser()
logging_chain = RunnableLambda(log_to_database)

# Compose them explicitly
rag_chain = (
    RunnableParallel(context=retrieval_chain, question=RunnablePassthrough())
    | generation_chain
)

# Side effects handled separately
result = rag_chain.invoke("What is LCEL?")
logging_chain.invoke(result)
```

### One Tool Per Capability

Each tool should encapsulate a single external capability. Avoid creating "god tools" that handle multiple unrelated operations.

```python
from langchain_core.tools import tool

# Good: focused, single-purpose tools
@tool
def search_web(query: str) -> str:
    """Search the web for current information about a topic."""
    return web_client.search(query)

@tool
def query_database(sql: str) -> str:
    """Execute a read-only SQL query against the analytics database."""
    return db_client.execute(sql)

@tool
def calculate(expression: str) -> str:
    """Evaluate a mathematical expression and return the result."""
    return str(eval_safe(expression))
```

---

## Open/Closed Principle (OCP)

### Pluggable Retrievers

Design retrieval pipelines that accept any retriever implementation without modifying the pipeline code.

```python
from langchain_core.retrievers import BaseRetriever

def build_rag_chain(retriever: BaseRetriever, llm, prompt):
    """Build a RAG chain that works with any retriever implementation."""
    return (
        RunnableParallel(
            context=retriever | RunnableLambda(format_docs),
            question=RunnablePassthrough(),
        )
        | prompt
        | llm
        | StrOutputParser()
    )

# Extend by plugging in different retrievers -- no modification needed
chroma_rag = build_rag_chain(chroma_retriever, llm, prompt)
pinecone_rag = build_rag_chain(pinecone_retriever, llm, prompt)
ensemble_rag = build_rag_chain(ensemble_retriever, llm, prompt)
```

### Pluggable Memory

```python
from langchain_core.chat_history import BaseChatMessageHistory

def build_conversational_chain(
    llm,
    prompt,
    get_session_history: Callable[[str], BaseChatMessageHistory],
):
    """Chain is open for extension via different history backends."""
    base_chain = prompt | llm | StrOutputParser()

    return RunnableWithMessageHistory(
        base_chain,
        get_session_history,
        input_messages_key="input",
        history_messages_key="history",
    )

# Extend with different backends
from langchain_community.chat_message_histories import (
    ChatMessageHistory,
    RedisChatMessageHistory,
    PostgresChatMessageHistory,
)

in_memory = build_conversational_chain(llm, prompt, lambda sid: ChatMessageHistory())
redis_backed = build_conversational_chain(
    llm, prompt, lambda sid: RedisChatMessageHistory(sid, url=REDIS_URL)
)
```

### Custom Runnables for Extension

```python
from langchain_core.runnables import Runnable, RunnableConfig
from typing import Any

class CachedRunnable(Runnable):
    """Wraps any Runnable with caching -- extends without modifying the original."""

    def __init__(self, runnable: Runnable, cache: dict):
        self.runnable = runnable
        self.cache = cache

    def invoke(self, input: Any, config: RunnableConfig | None = None) -> Any:
        cache_key = str(input)
        if cache_key in self.cache:
            return self.cache[cache_key]
        result = self.runnable.invoke(input, config)
        self.cache[cache_key] = result
        return result
```

---

## Liskov Substitution Principle (LSP)

### BaseChain and BaseTool Contracts

Any subclass of BaseTool must honor the contract: accept the declared input schema and return a string. Any Runnable must honor `.invoke()`, `.batch()`, `.stream()`, and their async counterparts.

```python
from langchain_core.tools import BaseTool
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="Search query string")
    max_results: int = Field(default=5, description="Maximum number of results")

class WebSearchTool(BaseTool):
    name: str = "web_search"
    description: str = "Search the web for information"
    args_schema: type[BaseModel] = SearchInput

    def _run(self, query: str, max_results: int = 5) -> str:
        results = self.client.search(query, limit=max_results)
        return "\n".join(r.snippet for r in results)

    async def _arun(self, query: str, max_results: int = 5) -> str:
        results = await self.client.asearch(query, limit=max_results)
        return "\n".join(r.snippet for r in results)

class InternalSearchTool(BaseTool):
    """Substitutable for WebSearchTool -- same contract, different implementation."""
    name: str = "internal_search"
    description: str = "Search internal knowledge base"
    args_schema: type[BaseModel] = SearchInput

    def _run(self, query: str, max_results: int = 5) -> str:
        results = self.vector_store.similarity_search(query, k=max_results)
        return "\n".join(r.page_content for r in results)
```

### Consistent Runnable Interface

Every LCEL component implements the Runnable protocol, making them interchangeable in chain composition:

```python
# All of these are Runnables and can be placed anywhere in a pipe
components = [
    ChatPromptTemplate.from_template("{input}"),  # Runnable
    ChatOpenAI(model="gpt-4o"),                   # Runnable
    StrOutputParser(),                             # Runnable
    RunnableLambda(lambda x: x.upper()),           # Runnable
    retriever,                                     # Runnable
]

# Any Runnable can be composed with any other Runnable
chain = components[0] | components[1] | components[2] | components[3]
```

---

## Interface Segregation Principle (ISP)

### Targeted Callback Handlers

Do not implement a single massive callback handler that handles every event. Create focused handlers for specific concerns.

```python
from langchain_core.callbacks import BaseCallbackHandler

class CostTrackingHandler(BaseCallbackHandler):
    """Only tracks token usage and cost -- ignores all other events."""

    def __init__(self):
        self.total_tokens = 0
        self.total_cost = 0.0

    def on_llm_end(self, response, **kwargs):
        usage = response.llm_output.get("token_usage", {})
        self.total_tokens += usage.get("total_tokens", 0)
        self.total_cost += self._calculate_cost(usage)

    def _calculate_cost(self, usage):
        prompt_cost = usage.get("prompt_tokens", 0) * 0.00001
        completion_cost = usage.get("completion_tokens", 0) * 0.00003
        return prompt_cost + completion_cost


class LatencyTrackingHandler(BaseCallbackHandler):
    """Only tracks timing -- ignores all other events."""

    def __init__(self):
        self.timings = []

    def on_llm_start(self, serialized, prompts, **kwargs):
        kwargs["start_time"] = time.time()

    def on_llm_end(self, response, **kwargs):
        elapsed = time.time() - kwargs.get("start_time", time.time())
        self.timings.append(elapsed)


class ErrorAlertHandler(BaseCallbackHandler):
    """Only handles errors -- sends alerts to monitoring."""

    def on_llm_error(self, error, **kwargs):
        alerting_service.send(f"LLM Error: {error}")

    def on_chain_error(self, error, **kwargs):
        alerting_service.send(f"Chain Error: {error}")

    def on_tool_error(self, error, **kwargs):
        alerting_service.send(f"Tool Error: {error}")
```

### Specific Retriever Interfaces

```python
from abc import ABC, abstractmethod

class SemanticRetriever(ABC):
    """Interface for pure semantic similarity retrieval."""
    @abstractmethod
    def similarity_search(self, query: str, k: int) -> list: ...

class MetadataFilterRetriever(ABC):
    """Interface for metadata-filtered retrieval."""
    @abstractmethod
    def filtered_search(self, query: str, filters: dict, k: int) -> list: ...

class HybridRetriever(SemanticRetriever, MetadataFilterRetriever):
    """Combines both interfaces only when both capabilities are needed."""
    pass
```

---

## Dependency Inversion Principle (DIP)

### Runnable Protocol as Abstraction

Depend on the Runnable protocol, not on concrete implementations. Accept any Runnable where an LLM, retriever, or chain is needed.

```python
from langchain_core.runnables import Runnable

class RAGService:
    """Depends on abstract Runnable interfaces, not concrete implementations."""

    def __init__(
        self,
        retriever: Runnable,
        llm: Runnable,
        prompt: Runnable,
        parser: Runnable,
    ):
        self.chain = (
            RunnableParallel(
                context=retriever | RunnableLambda(self._format_docs),
                question=RunnablePassthrough(),
            )
            | prompt
            | llm
            | parser
        )

    def _format_docs(self, docs):
        return "\n\n".join(d.page_content for d in docs)

    def answer(self, question: str) -> str:
        return self.chain.invoke(question)
```

### Configurable Components

```python
from langchain_core.runnables import ConfigurableField

# The chain depends on an abstract "llm" slot, not a specific model
llm = ChatOpenAI(model="gpt-4o-mini").configurable_alternatives(
    ConfigurableField(id="llm_provider"),
    default_key="openai",
    anthropic=ChatAnthropic(model="claude-sonnet-4-20250514"),
    local=ChatOllama(model="llama3"),
)

chain = prompt | llm | parser

# Swap implementation at runtime without changing chain structure
result = chain.invoke(input, config={"configurable": {"llm_provider": "anthropic"}})
```

### Chain Composition over Inheritance

Favor composing small Runnables over creating deep inheritance hierarchies:

```python
# Good: composition of independent Runnables
preprocessing = RunnableLambda(clean_input) | RunnableLambda(validate_input)
core_logic = prompt | llm | parser
postprocessing = RunnableLambda(format_output) | RunnableLambda(add_metadata)

full_pipeline = preprocessing | core_logic | postprocessing

# Bad: deep inheritance
class MySpecialChain(BaseChain):          # avoid
    class MyMoreSpecialChain(MySpecialChain):  # avoid
        pass
```

---

## Summary

| Principle | LangChain Application |
|-----------|----------------------|
| SRP | One chain per task, one tool per capability |
| OCP | Pluggable retrievers, memory backends, custom Runnables |
| LSP | Honor BaseTool/Runnable contracts, consistent interfaces |
| ISP | Focused callback handlers, specific retriever interfaces |
| DIP | Depend on Runnable protocol, use configurable components |

Applying these principles yields LangChain applications that are modular, testable, and maintainable at scale.
