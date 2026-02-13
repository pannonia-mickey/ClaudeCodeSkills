# Tool Creation Guide

Comprehensive reference for creating tools in LangChain, covering the @tool decorator, StructuredTool, BaseTool subclassing, async tools, error handling, tool descriptions, and dynamic tool selection.

---

## @tool Decorator

The simplest way to create a tool. The function's docstring becomes the tool description, and type hints define the input schema.

### Basic Usage

```python
from langchain_core.tools import tool

@tool
def search_wikipedia(query: str) -> str:
    """Search Wikipedia for information about a topic.

    Use this when you need factual, encyclopedic information about a subject.
    """
    import wikipedia
    try:
        return wikipedia.summary(query, sentences=3)
    except wikipedia.DisambiguationError as e:
        return f"Ambiguous query. Options: {', '.join(e.options[:5])}"
    except wikipedia.PageError:
        return f"No Wikipedia page found for '{query}'."
```

### With Type Annotations for Complex Inputs

```python
from typing import Optional

@tool
def search_documents(
    query: str,
    category: Optional[str] = None,
    max_results: int = 5,
    include_metadata: bool = False,
) -> str:
    """Search the internal document database.

    Args:
        query: The search query string.
        category: Optional category filter (e.g., 'engineering', 'sales', 'hr').
        max_results: Maximum number of results to return. Defaults to 5.
        include_metadata: Whether to include document metadata in results.
    """
    results = db.search(query, category=category, limit=max_results)
    if include_metadata:
        return "\n\n".join(
            f"[{r.metadata}]\n{r.content}" for r in results
        )
    return "\n\n".join(r.content for r in results)
```

### Returning Artifacts

Return both a string response (for the agent) and structured data (for the application):

```python
@tool(response_format="content_and_artifact")
def fetch_data(query: str) -> tuple[str, list[dict]]:
    """Fetch structured data from the analytics API.

    Returns a summary for the agent and raw data as an artifact.
    """
    data = analytics_api.query(query)
    summary = f"Found {len(data)} records. Top result: {data[0]['title']}"
    return summary, data
```

---

## StructuredTool

Provides more control than the decorator, with explicit Pydantic schemas.

### Basic StructuredTool

```python
from langchain_core.tools import StructuredTool
from pydantic import BaseModel, Field

class DatabaseQueryInput(BaseModel):
    sql: str = Field(description="The SQL query to execute. Must be SELECT only.")
    database: str = Field(
        default="analytics",
        description="Target database: 'analytics', 'users', or 'logs'",
    )
    timeout: int = Field(
        default=30,
        description="Query timeout in seconds",
        ge=1,
        le=120,
    )

def run_query(sql: str, database: str = "analytics", timeout: int = 30) -> str:
    if not sql.strip().upper().startswith("SELECT"):
        return "Error: Only SELECT queries are allowed."
    conn = get_connection(database)
    results = conn.execute(sql, timeout=timeout)
    return format_results(results)

query_tool = StructuredTool.from_function(
    func=run_query,
    name="database_query",
    description=(
        "Execute a read-only SQL query against the specified database. "
        "Use this for data analysis, aggregation, and lookup tasks."
    ),
    args_schema=DatabaseQueryInput,
    return_direct=False,
)
```

### With Async Support

```python
async def async_run_query(sql: str, database: str = "analytics", timeout: int = 30) -> str:
    if not sql.strip().upper().startswith("SELECT"):
        return "Error: Only SELECT queries are allowed."
    conn = await get_async_connection(database)
    results = await conn.execute(sql, timeout=timeout)
    return format_results(results)

query_tool = StructuredTool.from_function(
    func=run_query,
    coroutine=async_run_query,
    name="database_query",
    description="Execute a read-only SQL query against the specified database.",
    args_schema=DatabaseQueryInput,
)
```

---

## BaseTool Subclass

Maximum control for complex tools that need internal state, configuration, or lifecycle management.

### Full Implementation

```python
from langchain_core.tools import BaseTool
from pydantic import BaseModel, Field
from typing import Optional, Type

class APISearchInput(BaseModel):
    query: str = Field(description="Search query")
    filters: dict = Field(default_factory=dict, description="Optional filters")
    page: int = Field(default=1, ge=1, description="Result page number")

class APISearchTool(BaseTool):
    name: str = "api_search"
    description: str = (
        "Search the company's internal API for records. Supports filtering "
        "by department, date range, and status. Returns paginated results."
    )
    args_schema: Type[BaseModel] = APISearchInput

    # Tool configuration
    api_base_url: str = "https://api.internal.com"
    api_key: str = ""
    max_retries: int = 3

    def _run(
        self,
        query: str,
        filters: dict = None,
        page: int = 1,
    ) -> str:
        """Synchronous execution."""
        headers = {"Authorization": f"Bearer {self.api_key}"}
        params = {"q": query, "page": page, **(filters or {})}

        for attempt in range(self.max_retries):
            try:
                response = requests.get(
                    f"{self.api_base_url}/search",
                    params=params,
                    headers=headers,
                    timeout=10,
                )
                response.raise_for_status()
                data = response.json()
                return self._format_results(data)
            except requests.RequestException as e:
                if attempt == self.max_retries - 1:
                    return f"Error after {self.max_retries} attempts: {str(e)}"

    async def _arun(
        self,
        query: str,
        filters: dict = None,
        page: int = 1,
    ) -> str:
        """Async execution."""
        headers = {"Authorization": f"Bearer {self.api_key}"}
        params = {"q": query, "page": page, **(filters or {})}

        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.api_base_url}/search",
                params=params,
                headers=headers,
                timeout=aiohttp.ClientTimeout(total=10),
            ) as resp:
                data = await resp.json()
                return self._format_results(data)

    def _format_results(self, data: dict) -> str:
        results = data.get("results", [])
        if not results:
            return "No results found."
        formatted = []
        for r in results[:10]:
            formatted.append(f"- {r['title']}: {r['summary']}")
        total = data.get("total", len(results))
        return f"Found {total} results:\n" + "\n".join(formatted)
```

### Tool with State

```python
class ConversationMemoryTool(BaseTool):
    name: str = "memory"
    description: str = "Store or recall information from conversation memory."
    args_schema: Type[BaseModel] = MemoryInput

    memory_store: dict = Field(default_factory=dict)

    def _run(self, action: str, key: str, value: str = "") -> str:
        if action == "store":
            self.memory_store[key] = value
            return f"Stored: {key} = {value}"
        elif action == "recall":
            return self.memory_store.get(key, f"No memory found for key '{key}'")
        elif action == "list":
            return "\n".join(f"{k}: {v}" for k, v in self.memory_store.items())
        return f"Unknown action: {action}"
```

---

## Async Tools

### Async-Only Tools

```python
@tool
async def fetch_webpage(url: str) -> str:
    """Fetch and extract text content from a webpage.

    Use this when you need to read the contents of a specific URL.
    """
    async with aiohttp.ClientSession() as session:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=15)) as resp:
            if resp.status != 200:
                return f"Error: HTTP {resp.status}"
            html = await resp.text()
            soup = BeautifulSoup(html, "html.parser")
            for tag in soup(["script", "style", "nav", "footer"]):
                tag.decompose()
            text = soup.get_text(separator="\n", strip=True)
            return text[:3000]
```

### Running Sync Tools in Async Context

```python
import asyncio
from functools import partial

@tool
def cpu_intensive_analysis(data: str) -> str:
    """Perform CPU-intensive analysis on the provided data."""
    return heavy_computation(data)

# When using in async agent, LangChain automatically wraps sync tools
# But for explicit control:
async def run_sync_tool_async(tool_func, *args):
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, partial(tool_func, *args))
```

---

## Error Handling in Tools

### Comprehensive Error Handling Pattern

```python
@tool
def query_external_api(endpoint: str, method: str = "GET", body: str = "") -> str:
    """Query an external REST API.

    Args:
        endpoint: Full URL of the API endpoint.
        method: HTTP method (GET or POST).
        body: JSON request body for POST requests.
    """
    # Input validation
    if method not in ("GET", "POST"):
        return "Error: method must be GET or POST."

    if not endpoint.startswith("https://"):
        return "Error: Only HTTPS endpoints are supported for security."

    try:
        if method == "GET":
            response = requests.get(endpoint, timeout=10)
        else:
            try:
                parsed_body = json.loads(body) if body else {}
            except json.JSONDecodeError:
                return "Error: Invalid JSON in request body."
            response = requests.post(endpoint, json=parsed_body, timeout=10)

        response.raise_for_status()
        data = response.json()
        return json.dumps(data, indent=2)[:3000]

    except requests.Timeout:
        return "Error: Request timed out after 10 seconds. The API may be slow or unresponsive."
    except requests.ConnectionError:
        return f"Error: Could not connect to {endpoint}. Check the URL."
    except requests.HTTPError as e:
        return f"Error: API returned HTTP {e.response.status_code}: {e.response.text[:200]}"
    except Exception as e:
        return f"Error: Unexpected error: {type(e).__name__}: {str(e)}"
```

### Handle Parsing Errors in AgentExecutor

```python
executor = AgentExecutor(
    agent=agent,
    tools=tools,
    handle_parsing_errors=True,  # sends error back to LLM for self-correction
    # Or provide a custom handler:
    # handle_parsing_errors="Please format your tool call correctly.",
)
```

### Tool-Level Error Recovery

```python
from langchain_core.tools import ToolException

@tool(handle_tool_error=True)
def strict_tool(input: str) -> str:
    """A tool that raises ToolException for recoverable errors."""
    if not input.strip():
        raise ToolException("Input cannot be empty. Provide a valid query.")
    return process(input)

# Custom error handler
def handle_error(error: ToolException) -> str:
    return f"Tool failed: {str(error)}. Please adjust your input and try again."

@tool(handle_tool_error=handle_error)
def another_tool(input: str) -> str:
    """A tool with custom error handling."""
    ...
```

---

## Writing Effective Tool Descriptions

Tool descriptions are critical because the LLM uses them to decide when and how to use each tool.

### Description Guidelines

1. **State what the tool does** in the first sentence.
2. **Specify when to use it** and when NOT to use it.
3. **Describe input constraints** in the args docstring.
4. **Mention output format** so the agent knows what to expect.

### Good vs Bad Descriptions

```python
# Bad: vague, no guidance
@tool
def search(q: str) -> str:
    """Search for stuff."""
    ...

# Good: specific, actionable, with constraints
@tool
def search_product_catalog(query: str) -> str:
    """Search the product catalog by name, SKU, or description.

    Use this tool when the user asks about products, pricing, or availability.
    Do NOT use this for order status or customer information -- use the
    order_lookup or customer_search tools instead.

    Returns up to 5 matching products with name, price, and stock status.
    """
    ...
```

### Structured Argument Descriptions

```python
class SearchInput(BaseModel):
    query: str = Field(
        description="Natural language search query. Be specific -- "
        "'red running shoes size 10' works better than 'shoes'."
    )
    category: Optional[str] = Field(
        default=None,
        description="Product category to filter by. "
        "Options: 'electronics', 'clothing', 'home', 'sports'. "
        "Leave empty to search all categories.",
    )
    price_max: Optional[float] = Field(
        default=None,
        description="Maximum price in USD. Only set if user specifies a budget.",
    )
```

---

## Dynamic Tool Selection

### Runtime Tool Filtering

Give agents different tool sets based on context:

```python
def get_tools_for_user(user_role: str) -> list:
    """Return tools appropriate for the user's role."""
    base_tools = [search_tool, calculator_tool]

    if user_role == "admin":
        return base_tools + [database_write_tool, config_tool, delete_tool]
    elif user_role == "analyst":
        return base_tools + [database_read_tool, export_tool]
    else:
        return base_tools

# Build agent with role-specific tools
tools = get_tools_for_user(current_user.role)
agent = create_tool_calling_agent(llm, tools, prompt)
```

### Tool Selection with Retrieval

For large tool sets, retrieve the most relevant tools for each query:

```python
from langchain_core.vectorstores import InMemoryVectorStore

# Index tool descriptions
tool_descriptions = [
    {"name": t.name, "description": t.description}
    for t in all_tools
]

tool_vectorstore = InMemoryVectorStore.from_texts(
    [t["description"] for t in tool_descriptions],
    embeddings,
    metadatas=tool_descriptions,
)

def select_tools(query: str, k: int = 5) -> list:
    """Dynamically select the most relevant tools for a query."""
    results = tool_vectorstore.similarity_search(query, k=k)
    selected_names = {r.metadata["name"] for r in results}
    return [t for t in all_tools if t.name in selected_names]

# Use in agent
selected = select_tools(user_query)
agent = create_tool_calling_agent(llm, selected, prompt)
```

### Conditional Tool Availability

```python
@tool
def premium_analysis(data: str) -> str:
    """Perform advanced statistical analysis on data.

    This tool is only available to premium users.
    """
    return advanced_stats(data)

# Inject availability check
original_run = premium_analysis._run

def guarded_run(*args, **kwargs):
    if not current_user.is_premium:
        return "Error: This tool requires a premium subscription."
    return original_run(*args, **kwargs)

premium_analysis._run = guarded_run
```

---

## Tool Testing

```python
def test_search_tool():
    """Test tool with known inputs and expected outputs."""
    result = search_tool.invoke({"query": "test query"})
    assert isinstance(result, str)
    assert len(result) > 0

def test_tool_error_handling():
    """Verify tool handles errors gracefully."""
    result = query_tool.invoke({"sql": "DELETE FROM users"})
    assert "Error" in result
    assert "SELECT" in result  # should mention the constraint

def test_tool_schema():
    """Verify tool schema is correct."""
    schema = search_tool.args_schema.model_json_schema()
    assert "query" in schema["properties"]
    assert schema["properties"]["query"]["type"] == "string"
```
