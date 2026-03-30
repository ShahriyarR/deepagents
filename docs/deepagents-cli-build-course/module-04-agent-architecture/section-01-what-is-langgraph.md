# Section 1: What is LangGraph?

Understand why graph-based agents are more powerful than simple chains.

## The Chain Limitation

Traditional LLM applications use a simple **chain** pattern:

```
User → Prompt Template → LLM → Response
```

This works for simple tasks, but breaks down when:

- **Multiple tools** need to be called in sequence
- **Conditional logic** determines next steps
- **State** needs to persist between turns
- **Looping** is required until a goal is met

## Chains Are Linear

A chain processes input → produces output → done. There's no built-in way to:

- Loop back and call another tool
- Make decisions based on tool results
- Maintain conversation context across multiple turns

## Graphs Are Flexible

LangGraph uses a **graph** model where:

- **Nodes** are processing units (call model, execute tool, etc.)
- **Edges** connect nodes and define flow
- **State** flows through the graph and is updated by each node
- **Conditional edges** allow dynamic routing based on state

## Comparing Patterns

### Chain Pattern (LangChain Expression Language)

```python
# Simple chain: prompt → model → output
chain = prompt | model | output_parser
result = chain.invoke({"question": "..."})
```

### Graph Pattern (LangGraph)

```python
# Graph with state, nodes, and conditional routing
graph = StateGraph(AgentState)
graph.add_node("model", call_model)
graph.add_node("tools", execute_tools)
graph.add_edge("model", "tools", condition=should_continue)
# ... more nodes and edges
compiled = graph.compile()
```

## Why Graph-Based?

| Feature | Chains | Graphs |
|---------|--------|--------|
| Tool calling | Single tool | Multiple tools, loops |
| Conditional routing | Limited | Full flexibility |
| State management | None | Built-in state |
| Multi-turn memory | Manual | Automatic |
| Complex workflows | Hard | Natural fit |

## The Agent Loop Pattern

Most agents follow this pattern:

```
1. Model decides: Should I use a tool or respond?
2. If tool: Execute tool → Get result → Go to step 1
3. If respond: Generate text → Done (END)
```

This loop is naturally modeled as a graph:

```
┌──────────────────────────────────────────────┐
│                                              │
│  ┌───────┐    ┌─────────┐    ┌────────────┐  │
│  │ Model │───▶│ More    │───▶│   END      │  │
│  │ Node  │    │ Tools?  │    │            │  │
│  └───┬───┘    └────┬────┘    └────────────┘  │
│      │              │                         │
│      │ Yes         │ No                       │
│      ▼              ▼                         │
│  ┌────────┐    ┌─────────┐                    │
│  │ Tool   │───▶│  Model  │───────────────────┘
│  │ Node   │    │  Node   │
│  └────────┘    └─────────┘
│       │
└───────┴─────────────────────────── (loops back)
```

## LangGraph Core Concepts

### State

The `State` is a `TypedDict` that flows through the graph. Each node receives the current state and returns updates:

```python
class AgentState(TypedDict):
    messages: list[BaseMessage]  # Conversation history
    current_tool: str | None     # Last tool called
    iterations: int              # Loop counter
```

### Nodes

Nodes are Python functions that:
- Receive the current state
- Perform processing (call model, execute tool)
- Return state updates

```python
def model_node(state: AgentState) -> AgentState:
    # Call LLM with current messages
    response = llm.invoke(state["messages"])
    # Return updates to merge into state
    return {"messages": [response]}
```

### Edges

Edges connect nodes. Two types:

1. **Normal edges** — Always go from source to destination
2. **Conditional edges** — Route based on a function's output

```python
# Normal edge: model always goes to tools
graph.add_edge("model", "tools")

# Conditional edge: route based on model output
graph.add_edge("model", "tools", condition=should_continue)
```

### END Node

`END` is a special built-in node that marks completion. When the graph reaches `END`, execution stops.

## Your First LangGraph Agent

Here's a minimal complete example:

```python
from typing import TypedDict
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, END

# 1. Define state
class AgentState(TypedDict):
    messages: list

# 2. Create graph
builder = StateGraph(AgentState)

# 3. Add nodes
def model_node(state: AgentState):
    llm = ChatOpenAI(model="gpt-4o")
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

builder.add_node("model", model_node)

# 4. Add edges
builder.add_edge("__start__", "model")  # Start → Model
builder.add_edge("model", END)           # Model → END

# 5. Compile
graph = builder.compile()

# 6. Run
result = graph.invoke({"messages": [HumanMessage(content="Hello!")]})
```

This is the foundation. In the following sections, we'll build out tool calling and conditional routing.

## Key Takeaways

- **Chains are linear** — Input → Output, no loops or state
- **Graphs are flexible** — Nodes, edges, and state enable complex agents
- **State flows through the graph** — Each node updates state
- **Conditional edges** enable dynamic routing
- **END marks completion** — The graph stops when it reaches END

## Next Section

[Define State](./section-02-define-state.md) — Create your TypedDict state schema.
