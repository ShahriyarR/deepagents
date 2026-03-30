# Section 3: Build Tool Node

Create a node that executes tools and returns results to the graph.

## What is a Tool Node?

The **tool node** is the worker of your agent. When the model decides to call a tool, this node:

1. Receives the tool name and arguments from state
2. Executes the actual tool function
3. Returns the result to be added to messages

## Tool Node Signature

Every node follows the same pattern:

```python
def node_name(state: AgentState) -> AgentState:
    # Processing
    return {"key": value}  # Updates to merge into state
```

## Extracting Tool Calls

The model returns tool calls in its response. Extract them like this:

```python
from langchain_core.messages import AIMessage, ToolMessage

def tool_node(state: AgentState) -> AgentState:
    # Get the last message
    last_message = state["messages"][-1]
    
    # Check if model called tools
    if not hasattr(last_message, "tool_calls"):
        return {"messages": state["messages"]}
    
    # Execute each tool call
    tool_results = []
    for tool_call in last_message.tool_calls:
        tool_name = tool_call["name"]
        tool_args = tool_call["args"]
        
        # Execute the tool
        result = execute_tool(tool_name, tool_args)
        
        # Create tool message
        tool_message = ToolMessage(
            content=str(result),
            tool_call_id=tool_call["id"],
        )
        tool_results.append(tool_message)
    
    # Return updates
    return {"messages": state["messages"] + tool_results}
```

## Complete Tool Node Implementation

Here's a production-ready tool node:

```python
from typing import Literal
from langchain_core.messages import AIMessage, ToolMessage
from deepagents_cli.tools import TOOL_REGISTRY

class AgentState(TypedDict):
    messages: list[BaseMessage]
    iterations: int


def execute_tools(state: AgentState) -> AgentState:
    """Execute tools requested by the model and return results."""
    last_message = state["messages"][-1]
    
    # Guard: Only process if last message has tool calls
    if not isinstance(last_message, AIMessage) or not last_message.tool_calls:
        return {"messages": state["messages"]}
    
    new_messages: list[BaseMessage] = []
    
    # Execute each tool call
    for tool_call in last_message.tool_calls:
        tool_name = tool_call["name"]
        tool_args = tool_call["args"]
        tool_id = tool_call["id"]
        
        # Look up tool from registry
        if tool_name not in TOOL_REGISTRY:
            error_msg = f"Tool '{tool_name}' not found"
            new_messages.append(
                ToolMessage(content=error_msg, tool_call_id=tool_id)
            )
            continue
        
        tool = TOOL_REGISTRY[tool_name]
        
        try:
            # Execute tool with arguments
            result = tool.invoke(tool_args)
            content = str(result)
        except Exception as e:
            content = f"Error executing {tool_name}: {e}"
        
        # Create tool message with result
        new_messages.append(
            ToolMessage(content=content, tool_call_id=tool_id)
        )
    
    return {
        "messages": state["messages"] + new_messages,
        "iterations": state["iterations"] + 1,
    }
```

## Tool Registry Pattern

For clean tool management, use a registry:

```python
from langchain_core.tools import BaseTool

TOOL_REGISTRY: dict[str, BaseTool] = {}


def register_tool(tool: BaseTool) -> None:
    """Register a tool in the global registry."""
    TOOL_REGISTRY[tool.name] = tool


def get_tool(name: str) -> BaseTool | None:
    """Get a tool by name."""
    return TOOL_REGISTRY.get(name)


# Register built-in tools
from deepagents_cli.tools import ReadFileTool, WriteFileTool, ExecuteTool

register_tool(ReadFileTool())
register_tool(WriteFileTool())
register_tool(ExecuteTool())
```

## Handling Tool Errors

Always handle tool errors gracefully:

```python
def execute_tools(state: AgentState) -> AgentState:
    last_message = state["messages"][-1]
    
    if not isinstance(last_message, AIMessage) or not last_message.tool_calls:
        return {"messages": state["messages"]}
    
    new_messages = []
    
    for tool_call in last_message.tool_calls:
        tool_name = tool_call["name"]
        tool_id = tool_call["id"]
        
        try:
            tool = get_tool(tool_name)
            if tool is None:
                raise ValueError(f"Tool '{tool_name}' not found")
            
            result = tool.invoke(tool_call["args"])
            content = str(result)
            
        except Exception as e:
            # Return error as tool message — model will see it and can recover
            content = f"Error: {type(e).__name__}: {e}"
        
        new_messages.append(
            ToolMessage(content=content, tool_call_id=tool_id)
        )
    
    return {
        "messages": state["messages"] + new_messages,
        "iterations": state["iterations"] + 1,
    }
```

## The Tool Call Flow

Here's how tool execution fits into the full flow:

```
1. Model Node:
   - Receives: messages with user input
   - Decides: Call read_file tool
   - Returns: AIMessage with tool_calls=[{name: "read_file", args: {...}}]

2. Tool Node:
   - Receives: messages + AIMessage with tool_calls
   - Executes: read_file.invoke(args)
   - Returns: ToolMessage with file contents

3. Model Node Again:
   - Receives: messages + ToolMessage with results
   - Decides: Generate response or call more tools
   - Returns: Final response or more tool calls
```

## Adding Iteration Tracking

Prevent infinite loops with iteration counting:

```python
MAX_ITERATIONS = 10

def execute_tools(state: AgentState) -> AgentState:
    """Execute tools with iteration limiting."""
    iterations = state.get("iterations", 0)
    
    if iterations >= MAX_ITERATIONS:
        # Stop the loop by returning error message
        return {
            "messages": state["messages"] + [
                AIMessage(content="Maximum iterations reached. Please try a simpler request.")
            ]
        }
    
    # ... tool execution ...
    
    return {
        "messages": state["messages"] + new_messages,
        "iterations": iterations + 1,
    }
```

## Tool Result as String

Always convert tool results to strings for the model:

```python
# GOOD: Convert to string
result = tool.invoke(args)
content = str(result)

# Sometimes you need structured output preserved:
import json
content = json.dumps(result, indent=2)
```

## Testing Your Tool Node

```python
import pytest
from langchain_core.messages import AIMessage, HumanMessage

def test_tool_node_executes_tools():
    state = AgentState(
        messages=[
            HumanMessage(content="Read myfile.txt"),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "name": "read_file",
                        "args": {"file_path": "myfile.txt"},
                        "id": "call_123",
                    }
                ],
            ),
        ],
        iterations=0,
    )
    
    result = execute_tools(state)
    
    # Check tool message was added
    assert len(result["messages"]) == 3
    assert isinstance(result["messages"][-1], ToolMessage)
    assert "myfile.txt" in result["messages"][-1].content
    assert result["iterations"] == 1
```

## Key Takeaways

- **Tool nodes extract tool_calls** from the model's response
- **Execute tools via `invoke()`** — Don't call the function directly
- **Return ToolMessage** — The model needs results in this format
- **Handle errors gracefully** — Model can often recover
- **Increment iterations** — Prevents infinite loops
- **Convert results to strings** — Model expects string content

## Next Section

[Build Model Node](./section-04-build-model-node.md) — Create the node that calls the LLM and decides actions.
