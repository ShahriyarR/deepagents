# Section 2: Define State

Create the TypedDict schema that flows through your agent graph.

## Why State Matters

In LangGraph, **state** is the connective tissue between nodes. It holds:

- Conversation messages
- Tool call history
- Intermediate results
- Control flow flags

Each node receives the current state and returns updates that get merged into the state.

## TypedDict for Type Safety

LangGraph uses `TypedDict` to define state schema:

```python
from typing import TypedDict
```

`TypedDict` gives you:
- Dictionary access with type checking
- IDE autocomplete for state keys
- Runtime validation of state structure

## Basic State Schema

Here's a minimal agent state:

```python
from typing import TypedDict
from langchain_core.messages import BaseMessage

class AgentState(TypedDict):
    """State that flows through the agent graph."""
    
    messages: list[BaseMessage]
```

That's it! The graph will carry `messages` through all nodes.

## Extended State Schema

Most agents need more than just messages:

```python
from typing import TypedDict, NotRequired
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage

class AgentState(TypedDict):
    """Full agent state with tool calling support."""
    
    messages: list[BaseMessage]
    current_tool: str | None
    tool_results: dict[str, str]
    iterations: int
```

### What Each Field Does

| Field | Type | Purpose |
|-------|------|---------|
| `messages` | `list[BaseMessage]` | Full conversation history |
| `current_tool` | `str \| None` | Name of tool being called |
| `tool_results` | `dict[str, str]` | Results from tool executions |
| `iterations` | `int` | Loop counter to prevent infinite loops |

## Optional vs Required Fields

Use `NotRequired` for fields that may not always be present:

```python
from typing import TypedDict, NotRequired

class AgentState(TypedDict):
    messages: list[BaseMessage]
    error: NotRequired[str | None]  # May not exist
    user_context: NotRequired[dict]  # Optional context
```

Use `Required` explicitly (rarely needed since fields are required by default):

```python
from typing import TypedDict, Required

class StrictState(TypedDict):
    messages: Required[list]  # Must always be present
    optional_field: str       # Still required, no annotation needed
```

## Immutable Updates

LangGraph state updates are **immutable**. Nodes return updates, not modified state:

```python
# GOOD: Return updates to merge
def model_node(state: AgentState) -> AgentState:
    new_message = llm.invoke(state["messages"])
    return {"messages": state["messages"] + [new_message]}

# BAD: Don't modify state in place
def bad_model_node(state: AgentState) -> AgentState:
    response = llm.invoke(state["messages"])
    state["messages"].append(response)  # Mutates original!
    return state
```

LangGraph merges returned updates into the existing state.

## Incrementing Counters

To increment a counter, read the current value and return the new value:

```python
def tool_node(state: AgentState) -> AgentState:
    # ... execute tool ...
    return {"iterations": state["iterations"] + 1}
```

## Adding Messages

Messages are appended, not replaced:

```python
def model_node(state: AgentState) -> AgentState:
    response = llm.invoke(state["messages"])
    return {"messages": state["messages"] + [response]}
```

## Complete State Example

Here's a production-ready state schema:

```python
from typing import TypedDict, NotRequired
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage, ToolMessage

class AgentState(TypedDict):
    """Complete state for a tool-calling agent."""
    
    messages: list[BaseMessage]
    current_tool: NotRequired[str | None]
    tool_results: NotRequired[dict[str, str]]
    iterations: int
    user_id: NotRequired[str | None]
    session_id: NotRequired[str | None]
```

## Using State in Nodes

Here's how nodes access state:

```python
def model_node(state: AgentState) -> AgentState:
    """Call the LLM with current messages."""
    # Read from state
    messages = state["messages"]
    iterations = state.get("iterations", 0)
    
    # Make decision based on state
    if iterations > 10:
        return {
            "messages": messages + [AIMessage(content="Too many iterations")],
            "current_tool": None,
        }
    
    # Call model
    response = llm.invoke(messages)
    
    # Return updates
    return {
        "messages": messages + [response],
        "iterations": iterations + 1,
    }
```

## State Type Safety

LangGraph validates state at runtime. If a node returns an unknown key:

```python
# This will raise an error
def bad_node(state: AgentState) -> AgentState:
    return {"unknown_key": "value"}  # KeyError at runtime!
```

## Best Practices

1. **Keep state minimal** — Only store what nodes need
2. **Use `NotRequired`** for optional fields
3. **Return updates, not mutations** — Immutability prevents bugs
4. **Document field purposes** — Other developers need to understand state
5. **Use clear field names** — `tool_results` not `tr`

## Next Steps

Your state schema is the foundation. Next, we'll build:

1. **Tool Node** — Executes tools and returns results
2. **Model Node** — Calls the LLM and decides actions
3. **Router Logic** — Decides whether to call tools or end

## Key Takeaways

- **TypedDict** defines your state schema
- **State flows through all nodes** — Each node receives and updates it
- **Immutable updates** — Return new values, don't mutate
- **NotRequired** for optional fields
- **LangGraph validates** state structure at runtime
