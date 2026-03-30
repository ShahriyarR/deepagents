# Section 5: Create Graph

Wire nodes together with edges and compile the executable graph.

## The StateGraph Container

LangGraph uses `StateGraph` as the container for your agent:

```python
from langgraph.graph import StateGraph

# Initialize with your state schema
builder = StateGraph(AgentState)
```

## Adding Nodes

Use `add_node()` to add processing nodes:

```python
# Add our model node
builder.add_node("model", model_node)

# Add our tool execution node
builder.add_node("tools", execute_tools)
```

### Node Names

Choose clear, descriptive names:

| Name | Purpose |
|------|---------|
| `"model"` | Calls the LLM |
| `"tools"` | Executes tools |
| `"supervisor"` | Routes to other nodes |
| `"generate"` | Final response generation |

## Adding Edges

Edges connect nodes and define flow.

### Normal Edges

Always go from source to destination:

```python
# Start goes to model
builder.add_edge("__start__", "model")

# Model goes to tools (after tool node runs, flow continues)
builder.add_edge("model", "tools")

# Tools go back to model (loop)
builder.add_edge("tools", "model")
```

### The `__start__` Node

`__start__` is a special built-in node — the entry point. Connect it to your first node:

```python
builder.add_edge("__start__", "model")
```

## Adding Conditional Edges

Conditional edges route based on a function's return value:

```python
from langgraph.types import Command
from langchain_core.messages import AIMessage

def should_continue(state: AgentState) -> Literal["tools", "END"]:
    """Decide whether to call tools or end."""
    last_message = state["messages"][-1]
    
    # If model made tool calls, continue to tools
    if isinstance(last_message, AIMessage) and last_message.tool_calls:
        return "tools"
    
    # Otherwise, we're done
    return END
```

Then add the conditional edge:

```python
# After model, check if we should continue or end
builder.add_conditional_edges("model", should_continue)

# This creates routes: model → tools OR model → END
```

## The END Node

`END` is a special built-in node that marks completion. When the graph reaches `END`, execution stops.

```python
from langgraph.graph import END

# Explicit edge to END
builder.add_edge("model", END)

# Conditional edges ending at END
builder.add_conditional_edges("model", should_continue)
# should_continue returns "tools" → tools node
#                returns END → stops execution
```

## Complete Graph Construction

Here's the full pattern:

```python
from typing import Literal
from langgraph.graph import StateGraph, END
from langchain_openai import ChatOpenAI

# Define state (from earlier section)
class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    iterations: int


# Create nodes (from earlier sections)
def model_node(state: AgentState) -> AgentState:
    # ... calls LLM ...
    return {"messages": [response]}


def execute_tools(state: AgentState) -> AgentState:
    # ... executes tools ...
    return {"messages": tool_messages, "iterations": state["iterations"] + 1}


# Routing function
def should_continue(state: AgentState) -> Literal["tools", "END"]:
    last_message = state["messages"][-1]
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    return END


# Build the graph
builder = StateGraph(AgentState)

# Add nodes
builder.add_node("model", model_node)
builder.add_node("tools", execute_tools)

# Add edges
builder.add_edge("__start__", "model")           # Start → Model
builder.add_conditional_edges("model", should_continue)  # Model → Tools/END
builder.add_edge("tools", "model")               # Tools → Model (loop back)

# Compile
graph = builder.compile()
```

## Visualizing the Flow

```
                    ┌─────────────────────────────────────┐
                    │                                     │
                    ▼                                     │
┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────┐  │
│  START  │───▶│  Model   │───▶│ Should  │───▶│ END  │  │
└─────────┘    │  Node    │    │ Continue?│    └──────┘  │
               └────┬─────┘    └────┬────┘             │
                    │                │                   │
                    │           ┌────┴────┐              │
                    │           │         │              │
                    │           ▼         ▼              │
                    │      ┌────────┐ ┌────────┐         │
                    └─────▶│ Tool   │ │ Model  │─────────┘
                           │  Node  │ │  Node  │
                           └────────┘ └────────┘
                                │
                                └─────────────────────────┘
                                    (loops to model)
```

## Compilation

Compile the graph to get an executable:

```python
# Compile creates an immutable, thread-safe graph
graph = builder.compile()

# Optional: Add checkpointer for state persistence
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)
```

## Running the Graph

### Synchronous

```python
# Invoke with initial state
result = graph.invoke({
    "messages": [HumanMessage(content="Read myfile.txt")],
    "iterations": 0,
})

# Result contains final state
for message in result["messages"]:
    print(f"{message.type}: {message.content}")
```

### Asynchronous

```python
# Async invocation for concurrent operations
result = await graph.ainvoke({
    "messages": [HumanMessage(content="Read myfile.txt")],
    "iterations": 0,
})
```

## Streaming Output

Get intermediate results as they happen:

```python
# Stream through graph execution
for chunk in graph.stream({"messages": [HumanMessage(content="Hello")])}:
    print(chunk)
    # {'model': {'messages': [...]}}
    # {'tools': {'messages': [...]}}
    # {'model': {'messages': [...]}}
```

### Streaming Tokens

For LLM token streaming:

```python
# Use astream_events on the compiled graph
async for event in graph.astream_events(
    {"messages": [HumanMessage(content="Hello")]},
    version="v1",
):
    if event["event"] == "on_chat_model_stream":
        print(event["data"]["chunk"].content, end="", flush=True)
```

## Configuration

Pass configuration at runtime:

```python
# Thread ID for state persistence
config = {"configurable": {"thread_id": "user-123-sessions"}}

# Recursion limit (default: 25)
config = {"configurable": {"recursion_limit": 100}}

# Run with config
result = graph.invoke(initial_state, config=config)
```

## Complete Run Example

```python
import asyncio
from typing import Annotated, Literal
from langchain_openai import ChatOpenAI
from langchain_core.messages import AIMessage, BaseMessage, HumanMessage, ToolMessage
from langgraph.graph import StateGraph, END, add_messages


class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    iterations: int


async def main():
    # Build graph (same as before)
    builder = StateGraph(AgentState)
    builder.add_node("model", model_node)
    builder.add_node("tools", execute_tools)
    builder.add_edge("__start__", "model")
    builder.add_conditional_edges("model", should_continue)
    builder.add_edge("tools", "model")
    
    graph = builder.compile()
    
    # Run
    result = await graph.ainvoke({
        "messages": [HumanMessage(content="Read the file at src/main.py")],
        "iterations": 0,
    })
    
    # Print final response
    for msg in result["messages"]:
        if isinstance(msg, AIMessage) and not msg.tool_calls:
            print(msg.content)


if __name__ == "__main__":
    asyncio.run(main())
```

## Key Takeaways

- **StateGraph** is the container for your agent
- **add_node()** adds processing nodes
- **add_edge()** creates fixed routing
- **add_conditional_edges()** creates dynamic routing
- **should_continue** function returns next node name or END
- **compile()** builds the executable graph
- **invoke() / ainvoke()** runs the graph
- **stream() / astream()** gets intermediate results

## Next Section

[Quiz](./quiz.md) — Test your understanding of agent architecture.
