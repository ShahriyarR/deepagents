# Section 4: Build Model Node

Create the node that calls the LLM and decides whether to use tools or respond.

## What is a Model Node?

The **model node** is the brain of your agent. It:

1. Receives the current conversation (messages)
2. Calls the LLM with tool definitions
3. Returns the model's response (text and/or tool calls)

## Model Node with Tools

The key is binding tools to the model:

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import AIMessage
from langchain_core.utils.function_calling import convert_to_openai_function_schema

class AgentState(TypedDict):
    messages: list[BaseMessage]


llm = ChatOpenAI(model="gpt-4o")

# Bind tools to the model
llm_with_tools = llm.bind_tools(
    tools,  # List of LangChain tools
    tool_choice="auto",  # Let model decide which tool (or "none")
)


def call_model(state: AgentState) -> AgentState:
    """Call the LLM with current messages and tools."""
    messages = state["messages"]
    
    # Model responds with text and/or tool calls
    response = llm_with_tools.invoke(messages)
    
    return {"messages": [response]}
```

## Complete Model Node

Here's a production-ready model node:

```python
from typing import Annotated, Literal
from langchain_openai import ChatOpenAI
from langchain_core.language_models import BaseChatModel
from langchain_core.messages import AIMessage, BaseMessage, HumanMessage
from langgraph.graph import add_messages

class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]


def create_model_node(llm: BaseChatModel, tools: list) -> callable:
    """Factory to create a model node with bound tools."""
    
    # Bind tools to model
    llm_with_tools = llm.bind_tools(tools, tool_choice="auto")
    
    def call_model(state: AgentState) -> AgentState:
        """Invoke LLM with conversation history."""
        messages = state["messages"]
        
        # Guard: If last message is from model, don't call again
        # This prevents duplicate calls after tool results
        if messages and isinstance(messages[-1], AIMessage):
            # Check if model already responded (no tool calls and no text)
            last_msg = messages[-1]
            if not last_msg.tool_calls and not last_msg.content:
                return {"messages": []}
        
        response = llm_with_tools.invoke(messages)
        
        return {"messages": [response]}
    
    return call_model


# Usage
model_node = create_model_node(
    llm=ChatOpenAI(model="gpt-4o"),
    tools=[read_file_tool, write_file_tool, execute_tool],
)
```

## Using `add_messages` for List Updates

The `Annotated` type with `add_messages` makes list updates cleaner:

```python
from typing import Annotated
from langgraph.graph import add_messages

class AgentState(TypedDict):
    # When a node returns {"messages": [new_msg]}, LangGraph
    # automatically appends to the list instead of replacing it
    messages: Annotated[list[BaseMessage], add_messages]
```

Without `add_messages`, you'd need to manually append:

```python
# Without add_messages
return {"messages": state["messages"] + [response]}

# With add_messages
return {"messages": [response]}  # LangGraph appends automatically
```

## The `add_messages` Reducer

LangGraph's `add_messages` reducer handles:

- **Appending new messages** to the list
- **Merging duplicate tool results** (avoiding duplicates)
- **Preserving message order**

This makes nodes simpler — just return the new messages.

## Guard Against Duplicate Calls

After tools execute, the model is called again. Guard against infinite loops:

```python
def call_model(state: AgentState) -> AgentState:
    messages = state["messages"]
    
    # If last message is a ToolMessage, model should respond
    # If last message is AIMessage with no content and no tools, stop
    if len(messages) > 1:
        last = messages[-1]
        second_last = messages[-2]
        
        # If we just got tool results and model already responded
        if isinstance(last, AIMessage) and isinstance(second_last, ToolMessage):
            if not last.content and not last.tool_calls:
                return {"messages": []}  # No update = stop
```

## Model Decision Making

The model decides what to do based on:

1. **System prompt** — Tells the model its role and capabilities
2. **Tool definitions** — Schema of available tools
3. **Conversation history** — Previous messages and tool results

## System Prompt for Tool Use

Your system prompt should guide tool use:

```python
SYSTEM_PROMPT = """You are a helpful coding assistant.

You have access to tools to:
- Read files: read_file(path)
- Write files: write_file(path, content)
- Execute commands: execute(command)

When a user asks you to do something:
1. If you need to read a file, use read_file
2. If you need to write output, use write_file
3. If you need to run a command, use execute

Use tools when they help answer the user's question.
Be concise and direct in your responses.
"""
```

## Tool Choice Options

When binding tools, you can control tool selection:

```python
# "auto" — Model decides whether to use no tools, one tool, or multiple
llm.bind_tools(tools, tool_choice="auto")

# "none" — Model won't call any tools (text-only responses)
llm.bind_tools(tools, tool_choice="none")

# "any" — Model must call exactly one tool
llm.bind_tools(tools, tool_choice="any")

# Force specific tool
llm.bind_tools(tools, tool_choice={"type": "function", "function": {"name": "read_file"}})
```

## Model Node Without Tools

For simple text-only responses:

```python
def call_model(state: AgentState) -> AgentState:
    """Call LLM without tools (text-only mode)."""
    messages = state["messages"]
    
    # Plain model without tool binding
    response = llm.invoke(messages)
    
    return {"messages": [response]}
```

## Testing the Model Node

```python
import pytest
from langchain_core.messages import HumanMessage

def test_model_node_calls_llm():
    state = AgentState(
        messages=[HumanMessage(content="Hello!")],
    )
    
    # Mock LLM
    with patch("deepagents_cli.agent.ChatOpenAI") as mock_llm:
        mock_instance = MagicMock()
        mock_instance.invoke.return_value = AIMessage(content="Hi there!")
        mock_llm.return_value = mock_instance
        
        result = call_model(state)
    
    # Verify LLM was called
    mock_instance.invoke.assert_called_once()
    
    # Verify response was added to messages
    assert len(result["messages"]) == 2
    assert result["messages"][-1].content == "Hi there!"
```

## Complete Example

Here's a full model node with all features:

```python
from typing import Annotated
from langchain_openai import ChatOpenAI
from langchain_core.language_models import BaseChatModel
from langchain_core.messages import AIMessage, BaseMessage, HumanMessage
from langgraph.graph import add_messages

SYSTEM_PROMPT = """You are a helpful coding assistant with access to tools."""


class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]


def create_model_node(
    llm: BaseChatModel,
    tools: list,
    system_prompt: str = SYSTEM_PROMPT,
) -> callable:
    """Create a model node with tools and system prompt."""
    
    # Bind tools to model
    llm_with_tools = llm.bind_tools(tools, tool_choice="auto")
    
    def call_model(state: AgentState) -> AgentState:
        # Prepend system prompt to first message only
        messages = state["messages"]
        
        # Inject system message if not present
        if not any(isinstance(m, HumanMessage) for m in messages):
            # First message - add system prompt
            messages = [SystemMessage(content=system_prompt)] + list(messages)
        
        response = llm_with_tools.invoke(messages)
        
        return {"messages": [response]}
    
    return call_model


# Create the node
model_node = create_model_node(
    llm=ChatOpenAI(model="gpt-4o"),
    tools=[read_file_tool, write_file_tool],
)
```

## Key Takeaways

- **Model node calls the LLM** with conversation history
- **Bind tools** using `llm.bind_tools()` — Model knows available tools
- **Return new messages** — `add_messages` handles list updates
- **Guard against loops** — Check if model already responded
- **System prompt guides behavior** — Tell the model when to use tools

## Next Section

[Create Graph](./section-05-create-graph.md) — Wire everything together with StateGraph, edges, and compilation.
