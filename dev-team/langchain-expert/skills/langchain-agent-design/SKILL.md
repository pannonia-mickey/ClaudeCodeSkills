---
name: langchain-agent-design
triggers:
  - "LangChain agent"
  - "LangChain tool"
  - "agent executor"
  - "ReAct agent"
  - "LangGraph"
description: |
  This skill should be used when the task involves designing and building LangChain agents, creating custom tools, configuring agent executors, implementing ReAct or plan-and-execute patterns, building stateful workflows with LangGraph, orchestrating multi-agent systems, or adding human-in-the-loop controls.
---

# LangChain Agent Design

## Agent Types Overview

LangChain agents are systems where an LLM decides which actions to take and in what order. The framework provides multiple agent architectures for different use cases.

### ReAct Agent (Recommended Default)

The ReAct (Reasoning + Acting) pattern interleaves thinking and tool use. This is the most commonly used agent pattern.

To create a basic ReAct agent:

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.agents import create_tool_calling_agent, AgentExecutor

llm = ChatOpenAI(model="gpt-4o", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful research assistant. Use tools when needed."),
    MessagesPlaceholder("chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),
])

tools = [search_tool, calculator_tool, database_tool]

agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    max_iterations=10,
    handle_parsing_errors=True,
)

result = agent_executor.invoke({"input": "What is the population of France times 3?"})
```

### Structured Chat Agent

To build an agent that handles complex multi-argument tools:

```python
from langchain.agents import create_structured_chat_agent

agent = create_structured_chat_agent(llm, tools, structured_prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
```

## Tool Creation

### Using the @tool Decorator

To define simple tools quickly:

```python
from langchain_core.tools import tool

@tool
def search_database(query: str, table: str = "documents") -> str:
    """Search the database for records matching the query.

    Args:
        query: The search query string.
        table: The database table to search. Defaults to 'documents'.
    """
    results = db.search(table, query)
    return "\n".join(str(r) for r in results[:5])
```

### Using StructuredTool

To create tools with explicit Pydantic schemas:

```python
from langchain_core.tools import StructuredTool
from pydantic import BaseModel, Field

class WeatherInput(BaseModel):
    city: str = Field(description="City name")
    units: str = Field(default="celsius", description="Temperature units: celsius or fahrenheit")

def get_weather(city: str, units: str = "celsius") -> str:
    data = weather_api.get(city, units)
    return f"{city}: {data['temp']}° {units}, {data['condition']}"

weather_tool = StructuredTool.from_function(
    func=get_weather,
    name="get_weather",
    description="Get current weather for a city",
    args_schema=WeatherInput,
)
```

### Async Tools

To create tools that support async execution:

```python
@tool
async def async_search(query: str) -> str:
    """Search the web asynchronously."""
    async with aiohttp.ClientSession() as session:
        async with session.get(f"https://api.search.com/search?q={query}") as resp:
            data = await resp.json()
            return "\n".join(r["snippet"] for r in data["results"][:5])
```

### Tool Error Handling

To make tools resilient and informative when they fail:

```python
@tool
def query_api(endpoint: str, params: str) -> str:
    """Query an external API endpoint with the given parameters."""
    try:
        response = requests.get(endpoint, params=json.loads(params), timeout=10)
        response.raise_for_status()
        return response.text[:2000]
    except requests.Timeout:
        return "Error: The API request timed out. Try a simpler query."
    except requests.HTTPError as e:
        return f"Error: API returned status {e.response.status_code}. Check the endpoint."
    except json.JSONDecodeError:
        return "Error: Invalid JSON in params. Provide valid JSON string."
```

## Agent Executor Configuration

To configure the agent executor for production use:

```python
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=False,                    # disable in production
    max_iterations=15,                # prevent infinite loops
    max_execution_time=60,            # timeout in seconds
    handle_parsing_errors=True,       # graceful error recovery
    return_intermediate_steps=True,   # for debugging and logging
    early_stopping_method="generate", # let LLM generate final answer on timeout
)
```

### Streaming Agent Output

To stream agent reasoning and tool calls:

```python
async for event in agent_executor.astream_events(
    {"input": "Analyze the latest sales data"},
    version="v2",
):
    kind = event["event"]
    if kind == "on_chat_model_stream":
        content = event["data"]["chunk"].content
        if content:
            print(content, end="", flush=True)
    elif kind == "on_tool_start":
        print(f"\n[Using tool: {event['name']}]")
    elif kind == "on_tool_end":
        print(f"[Tool result: {event['data'].output[:100]}]")
```

## LangGraph for Stateful Agents

LangGraph provides fine-grained control over agent behavior through explicit state machines.

### Basic LangGraph Agent

To build a stateful agent with LangGraph:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode, tools_condition
from typing import TypedDict, Annotated
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]

def call_model(state: AgentState):
    messages = state["messages"]
    response = llm_with_tools.invoke(messages)
    return {"messages": [response]}

# Build the graph
graph = StateGraph(AgentState)

graph.add_node("agent", call_model)
graph.add_node("tools", ToolNode(tools))

graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", tools_condition)
graph.add_edge("tools", "agent")

app = graph.compile()

result = app.invoke({"messages": [("human", "What's the weather in Paris?")]})
```

### Adding Persistence

To make agent state persist across conversations:

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)

# Each thread_id maintains separate conversation state
config = {"configurable": {"thread_id": "user-123"}}
result = app.invoke({"messages": [("human", "Hello")]}, config=config)

# Continue the same conversation
result = app.invoke({"messages": [("human", "What did I just say?")]}, config=config)
```

## Multi-Agent Patterns

### Supervisor Agent

To orchestrate multiple specialized agents with a supervisor:

```python
from langgraph.graph import StateGraph, START, END

class SupervisorState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    next_agent: str

def supervisor(state: SupervisorState):
    """Decide which agent should handle the next step."""
    response = supervisor_llm.with_structured_output(RouteDecision).invoke(
        state["messages"]
    )
    return {"next_agent": response.next_agent}

def route_to_agent(state: SupervisorState):
    return state["next_agent"]

graph = StateGraph(SupervisorState)
graph.add_node("supervisor", supervisor)
graph.add_node("researcher", researcher_agent)
graph.add_node("writer", writer_agent)
graph.add_node("reviewer", reviewer_agent)

graph.add_edge(START, "supervisor")
graph.add_conditional_edges("supervisor", route_to_agent, {
    "researcher": "researcher",
    "writer": "writer",
    "reviewer": "reviewer",
    "FINISH": END,
})
graph.add_edge("researcher", "supervisor")
graph.add_edge("writer", "supervisor")
graph.add_edge("reviewer", "supervisor")

multi_agent = graph.compile()
```

### Agent Handoff

To implement direct agent-to-agent handoff:

```python
def researcher_node(state):
    result = researcher_chain.invoke(state["messages"])
    return {
        "messages": [result],
        "next_agent": "writer" if result.content.startswith("RESEARCH COMPLETE") else "researcher",
    }
```

## Human-in-the-Loop

### Interrupt for Approval

To pause agent execution and wait for human approval before executing tools:

```python
from langgraph.graph import StateGraph

graph = StateGraph(AgentState)
graph.add_node("agent", call_model)
graph.add_node("tools", ToolNode(tools))

graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", tools_condition)
graph.add_edge("tools", "agent")

# Compile with interrupt_before to pause before tool execution
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["tools"],
)

# First invocation pauses before tools
result = app.invoke({"messages": [("human", "Delete all records")]}, config)
# result shows pending tool call

# Review and approve
app.invoke(None, config)  # resume execution

# Or reject by modifying state
app.update_state(config, {"messages": [("human", "Do not delete. Search instead.")]})
app.invoke(None, config)
```

### Human Feedback Node

To add a node that collects human feedback:

```python
def human_review(state: AgentState):
    """Pause for human review of the agent's proposed action."""
    last_message = state["messages"][-1]
    print(f"\nAgent wants to: {last_message.content}")
    feedback = input("Approve (y/n) or provide guidance: ")

    if feedback.lower() == "y":
        return {"messages": []}
    else:
        return {"messages": [("human", f"User feedback: {feedback}")]}
```

## Design Guidelines

1. **Start simple** -- begin with a ReAct agent and AgentExecutor before reaching for LangGraph.
2. **Write precise tool descriptions** -- the agent relies on descriptions to decide when and how to use each tool. Vague descriptions lead to wrong tool selection.
3. **Limit the tool set** -- give agents only the tools they need. More tools means more confusion.
4. **Set iteration limits** -- always configure `max_iterations` and `max_execution_time` to prevent runaway agents.
5. **Return intermediate steps** -- enable `return_intermediate_steps` during development to debug agent reasoning.
6. **Use LangGraph for complex flows** -- when the agent needs conditional logic, persistent state, or multi-agent coordination, use LangGraph instead of chaining AgentExecutors.
7. **Add human-in-the-loop for destructive actions** -- any tool that modifies state (database writes, file deletions, API mutations) should have approval gates.

## References

For detailed agent architecture patterns including ReAct, Plan-and-Execute, self-ask, multi-agent orchestration, LangGraph StateGraph, persistent memory, and streaming, see [agent-patterns.md](references/agent-patterns.md).

For comprehensive tool creation guidance including the @tool decorator, StructuredTool, BaseTool subclassing, async tools, error handling, tool descriptions, and dynamic tool selection, see [tool-creation.md](references/tool-creation.md).
