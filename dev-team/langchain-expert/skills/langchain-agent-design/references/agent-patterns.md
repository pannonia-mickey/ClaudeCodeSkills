# Agent Architecture Patterns

This reference covers the major agent architecture patterns in LangChain, from simple ReAct agents to complex multi-agent orchestration with LangGraph.

---

## ReAct Pattern

ReAct (Reasoning + Acting) is the standard agent pattern where the model alternates between reasoning about what to do and taking actions via tools.

### Core Implementation

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.agents import create_tool_calling_agent, AgentExecutor

llm = ChatOpenAI(model="gpt-4o", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", (
        "You are a research assistant. Think step by step before using tools. "
        "Always verify your findings with multiple sources when possible."
    )),
    MessagesPlaceholder("chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),
])

agent = create_tool_calling_agent(llm, tools, prompt)

executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    max_iterations=10,
    handle_parsing_errors=True,
    return_intermediate_steps=True,
)
```

### ReAct with LangGraph

```python
from langgraph.prebuilt import create_react_agent

# Simplest way to create a ReAct agent with LangGraph
react_agent = create_react_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=tools,
    state_modifier="You are a helpful research assistant.",
)

result = react_agent.invoke({"messages": [("human", "Research quantum computing")]})
```

### When to Use

ReAct works well for general-purpose tool-using agents, research tasks, question answering with tools, and any scenario where the agent needs to iteratively reason and act.

---

## Plan-and-Execute Pattern

The agent first creates a complete plan, then executes each step. Better for complex multi-step tasks that benefit from upfront planning.

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated, Sequence
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages

class PlanExecuteState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    plan: list[str]
    current_step: int
    results: list[str]

def planner(state: PlanExecuteState):
    """Create a step-by-step plan for the task."""
    plan_prompt = ChatPromptTemplate.from_template(
        "Create a numbered step-by-step plan to accomplish this task. "
        "Return only the steps, one per line.\n\nTask: {task}"
    )
    chain = plan_prompt | llm | StrOutputParser()
    task = state["messages"][-1].content
    plan_text = chain.invoke({"task": task})
    steps = [s.strip() for s in plan_text.strip().split("\n") if s.strip()]
    return {"plan": steps, "current_step": 0, "results": []}

def executor(state: PlanExecuteState):
    """Execute the current step of the plan."""
    step = state["plan"][state["current_step"]]
    context = "\n".join(state["results"]) if state["results"] else "No previous results."

    exec_prompt = ChatPromptTemplate.from_messages([
        ("system", f"Execute this step. Previous results:\n{context}"),
        ("human", step),
        MessagesPlaceholder("agent_scratchpad"),
    ])

    agent = create_tool_calling_agent(llm, tools, exec_prompt)
    result = AgentExecutor(agent=agent, tools=tools).invoke({"input": step})

    return {
        "results": state["results"] + [result["output"]],
        "current_step": state["current_step"] + 1,
    }

def should_continue(state: PlanExecuteState):
    if state["current_step"] >= len(state["plan"]):
        return "synthesize"
    return "executor"

def synthesizer(state: PlanExecuteState):
    """Combine all step results into a final answer."""
    all_results = "\n\n".join(
        f"Step {i+1}: {r}" for i, r in enumerate(state["results"])
    )
    synthesis = llm.invoke(
        f"Synthesize these results into a coherent answer:\n\n{all_results}"
    )
    return {"messages": [synthesis]}

graph = StateGraph(PlanExecuteState)
graph.add_node("planner", planner)
graph.add_node("executor", executor)
graph.add_node("synthesize", synthesizer)

graph.add_edge(START, "planner")
graph.add_edge("planner", "executor")
graph.add_conditional_edges("executor", should_continue)
graph.add_edge("synthesize", END)

plan_execute_agent = graph.compile()
```

### When to Use

Plan-and-Execute works best for complex research tasks, multi-step workflows with dependencies between steps, and tasks where having an explicit plan improves coherence and reduces wasted tool calls.

---

## Self-Ask Pattern

The agent decomposes complex questions into simpler sub-questions, answers each sub-question, then synthesizes the final answer.

```python
class SelfAskState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    sub_questions: list[str]
    sub_answers: list[str]
    final_answer: str

def decompose(state: SelfAskState):
    """Break the question into sub-questions."""
    decompose_chain = (
        ChatPromptTemplate.from_template(
            "Break this complex question into 2-4 simpler sub-questions "
            "that can be answered independently. One per line.\n\n"
            "Question: {question}"
        )
        | llm
        | StrOutputParser()
    )
    question = state["messages"][-1].content
    sub_qs = decompose_chain.invoke({"question": question})
    return {"sub_questions": [q.strip() for q in sub_qs.split("\n") if q.strip()]}

def answer_sub_question(state: SelfAskState):
    """Answer the next unanswered sub-question."""
    idx = len(state["sub_answers"])
    sub_q = state["sub_questions"][idx]

    answer = react_agent.invoke({"messages": [("human", sub_q)]})
    return {"sub_answers": state["sub_answers"] + [answer["messages"][-1].content]}

def check_sub_questions(state: SelfAskState):
    if len(state["sub_answers"]) < len(state["sub_questions"]):
        return "answer"
    return "synthesize"

def synthesize_answer(state: SelfAskState):
    """Combine sub-answers into final answer."""
    pairs = "\n".join(
        f"Q: {q}\nA: {a}"
        for q, a in zip(state["sub_questions"], state["sub_answers"])
    )
    original = state["messages"][-1].content
    final = llm.invoke(
        f"Based on these sub-answers, answer the original question.\n\n"
        f"Original: {original}\n\nSub-answers:\n{pairs}"
    )
    return {"messages": [final], "final_answer": final.content}
```

---

## Multi-Agent Orchestration

### Supervisor Pattern

A supervisor agent routes tasks to specialized worker agents.

```python
from pydantic import BaseModel, Field
from typing import Literal

class RouteDecision(BaseModel):
    next: Literal["researcher", "coder", "writer", "FINISH"]
    reasoning: str = Field(description="Why this agent was selected")

supervisor_llm = ChatOpenAI(model="gpt-4o").with_structured_output(RouteDecision)

def supervisor(state):
    system = (
        "You are a supervisor managing a team of agents: researcher, coder, writer. "
        "Based on the conversation, decide which agent should act next, or FINISH if done."
    )
    decision = supervisor_llm.invoke(
        [("system", system)] + state["messages"]
    )
    return {"next_agent": decision.next}

# Build worker agents as separate graphs or chains
researcher = create_react_agent(llm, research_tools, "You are a research specialist.")
coder = create_react_agent(llm, coding_tools, "You are a coding specialist.")
writer_agent = create_react_agent(llm, writing_tools, "You are a writing specialist.")

graph = StateGraph(SupervisorState)
graph.add_node("supervisor", supervisor)
graph.add_node("researcher", researcher)
graph.add_node("coder", coder)
graph.add_node("writer", writer_agent)

graph.add_edge(START, "supervisor")
graph.add_conditional_edges("supervisor", lambda s: s["next_agent"], {
    "researcher": "researcher",
    "coder": "coder",
    "writer": "writer",
    "FINISH": END,
})
for worker in ["researcher", "coder", "writer"]:
    graph.add_edge(worker, "supervisor")
```

### Hierarchical Multi-Agent

For complex organizations, nest supervisor-worker patterns:

```python
# Team lead agents, each managing their own sub-agents
research_team = build_research_team()   # supervisor + searcher + analyst
engineering_team = build_eng_team()     # supervisor + coder + tester

# Top-level supervisor
top_supervisor = StateGraph(TopState)
top_supervisor.add_node("coordinator", coordinator_node)
top_supervisor.add_node("research_team", research_team)
top_supervisor.add_node("engineering_team", engineering_team)
```

---

## Human-in-the-Loop Patterns

### Interrupt Before Tool Execution

```python
app = graph.compile(
    checkpointer=MemorySaver(),
    interrupt_before=["tools"],  # pause before any tool call
)

# First call pauses
result = app.invoke(input, config)
# Inspect pending tool calls in result

# Approve by resuming
app.invoke(None, config)

# Or reject by updating state
app.update_state(config, {
    "messages": [("human", "Don't do that. Instead, search for X.")]
})
app.invoke(None, config)
```

### Selective Interruption

Interrupt only for specific high-risk tools:

```python
def should_interrupt(state):
    last = state["messages"][-1]
    if hasattr(last, "tool_calls"):
        dangerous_tools = {"delete_records", "send_email", "modify_config"}
        for call in last.tool_calls:
            if call["name"] in dangerous_tools:
                return "human_review"
    return "tools"

graph.add_conditional_edges("agent", should_interrupt, {
    "human_review": "human_review",
    "tools": "tools",
})
```

### Approval with Timeout

```python
import asyncio

async def human_review_with_timeout(state, timeout=300):
    """Wait for human approval with a timeout."""
    try:
        approval = await asyncio.wait_for(
            get_human_input(state),
            timeout=timeout,
        )
        return {"approved": approval}
    except asyncio.TimeoutError:
        return {"approved": False, "reason": "Timed out waiting for approval"}
```

---

## LangGraph StateGraph Deep Dive

### Custom State Management

```python
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class ComplexState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    context: dict                    # accumulated context
    iteration: int                   # loop counter
    confidence: float                # quality metric
    status: str                      # workflow status

def check_quality(state: ComplexState):
    """Decide whether to iterate or finalize."""
    if state["confidence"] >= 0.9:
        return "finalize"
    if state["iteration"] >= 3:
        return "finalize"
    return "refine"
```

### Subgraph Composition

```python
# Define a subgraph for a specific task
research_subgraph = StateGraph(ResearchState)
research_subgraph.add_node("search", search_node)
research_subgraph.add_node("analyze", analyze_node)
research_subgraph.add_edge(START, "search")
research_subgraph.add_edge("search", "analyze")
research_subgraph.add_edge("analyze", END)
compiled_research = research_subgraph.compile()

# Use it as a node in the parent graph
parent_graph = StateGraph(ParentState)
parent_graph.add_node("research", compiled_research)
parent_graph.add_node("write", write_node)
parent_graph.add_edge(START, "research")
parent_graph.add_edge("research", "write")
parent_graph.add_edge("write", END)
```

---

## Persistent Memory

### Conversation Persistence with Checkpointing

```python
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.postgres import PostgresSaver

# SQLite for development
checkpointer = SqliteSaver.from_conn_string("checkpoints.db")

# Postgres for production
checkpointer = PostgresSaver.from_conn_string("postgresql://user:pass@host/db")

app = graph.compile(checkpointer=checkpointer)

# Each thread maintains its own state
config = {"configurable": {"thread_id": "conversation-abc"}}
```

### Long-Term Memory with Vector Store

```python
def agent_with_memory(state):
    """Agent node that reads/writes long-term memory."""
    query = state["messages"][-1].content

    # Recall relevant memories
    memories = memory_vectorstore.similarity_search(query, k=3)
    memory_context = "\n".join(m.page_content for m in memories)

    response = llm.invoke(
        [("system", f"Relevant memories:\n{memory_context}")] + state["messages"]
    )

    # Store new memory
    memory_vectorstore.add_texts(
        [response.content],
        metadatas=[{"timestamp": datetime.utcnow().isoformat()}],
    )

    return {"messages": [response]}
```

---

## Streaming Agent Output

### Full Event Streaming

```python
async for event in app.astream_events(
    {"messages": [("human", "Research and write about AI")]},
    config=config,
    version="v2",
):
    kind = event["event"]

    if kind == "on_chat_model_stream":
        token = event["data"]["chunk"].content
        if token:
            yield {"type": "token", "content": token}

    elif kind == "on_tool_start":
        yield {"type": "tool_start", "name": event["name"], "input": event["data"].get("input")}

    elif kind == "on_tool_end":
        yield {"type": "tool_end", "name": event["name"], "output": event["data"].output}

    elif kind == "on_chain_end" and event["name"] == "agent":
        yield {"type": "agent_step_complete"}
```

### Streaming State Updates

```python
async for state_update in app.astream(
    {"messages": [("human", "Complex task")]},
    config=config,
    stream_mode="updates",
):
    node_name = list(state_update.keys())[0]
    print(f"Node '{node_name}' updated state: {state_update[node_name]}")
```

---

## Pattern Selection Guide

| Pattern | Complexity | Best For |
|---------|-----------|----------|
| ReAct | Low | General tool-using agents |
| Plan-and-Execute | Medium | Complex multi-step tasks |
| Self-Ask | Medium | Complex questions requiring decomposition |
| Supervisor Multi-Agent | High | Diverse tasks requiring specialization |
| Hierarchical Multi-Agent | Very High | Large-scale organizational workflows |
| Human-in-the-Loop | Medium | High-stakes or destructive actions |

Start with ReAct for most use cases. Escalate to LangGraph when the agent needs explicit state management, conditional flows, or multi-agent coordination.
