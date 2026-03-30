# Section 5: Wire Tools to Model

Connect your tools to the LangChain model for tool calling.

## How Tool Calling Works in LangChain

When tools are bound to a model:

1. Model receives messages + tool schemas
2. Model decides whether to respond or call a tool
3. If tool call: returns `ToolMessage` with tool name + arguments
4. Framework executes tool and returns result
5. Model receives result and generates response

```
┌──────────────┐
│    Human     │
│   Message    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│    Model     │◀──── Tools Schema
│  (with tools │
│   bound)     │
└──────┬───────┘
       │
       ├─── Tool Call ───┐
       │                 │
       ▼                 ▼
┌──────────────┐  ┌──────────────┐
│  ToolMessage │  │    AIMessage │
│  (result)    │  │  (response)  │
└──────┬───────┘  └──────────────┘
       │                 │
       └────────┬────────┘
                ▼
         ┌──────────────┐
         │    Model     │
         │ (final resp) │
         └──────────────┘
```

## Bind Tools to Model

Update `src/deepagents_cli/models.py`:

```python
"""Model factory for creating LLM instances."""

from __future__ import annotations

from typing import Optional, Union

from langchain_openai import ChatOpenAI
from langchain_core.language_models import BaseChatModel
from langchain_core.tools import BaseTool


def create_model(
    model: Optional[str] = None,
    api_key: Optional[str] = None,
    tools: Optional[list[Union[BaseTool, type]]] = None,
) -> BaseChatModel:
    """Create a ChatOpenAI model instance.
    
    Args:
        model: Model name (e.g., "gpt-4o").
               Defaults to Config.default_model.
        api_key: OpenAI API key. Defaults to Config.openai_api_key.
        tools: Optional list of tools to bind to the model.
    
    Returns:
        Configured ChatOpenAI instance with tools bound.
    """
    from deepagents_cli.config import config
    
    llm = ChatOpenAI(
        model=model or config.default_model,
        api_key=api_key or config.openai_api_key,
        timeout=30,
    )
    
    if tools:
        return llm.bind_tools(tools)
    
    return llm
```

## Create a Tool-Calling REPL

Create `src/deepagents_cli/repl_with_tools.py`:

```python
"""Async REPL with tool calling support."""

from __future__ import annotations

import asyncio
from typing import Any

from langchain_core.messages import HumanMessage, ToolMessage
from langchain_core.tools import BaseTool

from deepagents_cli.models import create_model
from deepagents_cli.tools import read_file_tool, write_file, execute


def get_tools() -> list[BaseTool]:
    """Get all available tools."""
    return [read_file_tool, write_file, execute]


async def process_with_tools(user_input: str) -> str:
    """Process a message with tool calling.
    
    Args:
        user_input: User's message.
    
    Returns:
        AI response.
    """
    model = create_model(tools=get_tools())
    
    messages = [HumanMessage(content=user_input)]
    
    while True:
        response = model.invoke(messages)
        
        # Check if model wants to call a tool
        if hasattr(response, "tool_calls") and response.tool_calls:
            for tool_call in response.tool_calls:
                tool_name = tool_call["name"]
                tool_args = tool_call["args"]
                
                # Find the tool
                tool = None
                for t in get_tools():
                    if t.name == tool_name:
                        tool = t
                        break
                
                if not tool:
                    result = f"Error: Unknown tool '{tool_name}'"
                else:
                    # Execute the tool
                    try:
                        result = tool.invoke(tool_args)
                    except Exception as e:
                        result = f"Error executing tool: {e}"
                
                # Add tool result to messages
                messages.append(
                    ToolMessage(
                        content=result,
                        tool_call_id=tool_call["id"],
                    )
                )
        else:
            # No tool call, return the response
            return response.content


async def repl() -> int:
    """Run the REPL with tool calling."""
    print("Deep Agents CLI with Tools - type 'exit' to quit")
    print("Available tools: read_file, write_file, execute")
    print("-" * 50)
    
    while True:
        try:
            user_input = await asyncio.to_thread(input, ">>> ")
        except (EOFError, KeyboardInterrupt):
            print("\nGoodbye!")
            break
        
        if user_input.strip().lower() in ("exit", "quit", "q"):
            print("Goodbye!")
            break
        
        if not user_input.strip():
            continue
        
        try:
            response = await process_with_tools(user_input)
            print(response)
        except Exception as e:
            print(f"Error: {e}")
    
    return 0
```

## Test Tool Calling

Run the REPL:

```bash
uv run python -c "
import asyncio
from deepagents_cli.repl_with_tools import repl
asyncio.run(repl())
"
```

Try these inputs:

```
>>> Read the file at /tmp/test_file.txt
[Tool is called with file_path="/tmp/test_file.txt"]
[File contents displayed]

>>> Write "Hello from the agent!" to /tmp/agent_test.txt
[Tool is called]
File created successfully.

>>> Run "ls -la /tmp"
[Tool is called]
[Directory listing displayed]
```

## How the LLM Decides to Use Tools

The LLM sees your tools as a schema:

```python
{
  "name": "read_file",
  "description": "Read the contents of a file...",
  "parameters": {
    "type": "object",
    "properties": {
      "file_path": {
        "type": "string",
        "description": "Path to the file to read"
      },
      "max_lines": {
        "type": "number",
        "description": "Maximum number of lines to read"
      }
    },
    "required": ["file_path"]
  }
}
```

The LLM uses the description to decide when to call each tool. Good descriptions = better tool selection.

## Verbose Mode for Debugging

Add verbose logging to see what's happening:

```python
"""REPL with verbose debugging."""

import asyncio
import logging
from typing import Any

from langchain_core.messages import HumanMessage, ToolMessage
from langchain_core.tools import BaseTool

from deepagents_cli.models import create_model
from deepagents_cli.tools import read_file_tool, write_file, execute

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)


def get_tools() -> list[BaseTool]:
    """Get all available tools."""
    return [read_file_tool, write_file, execute]


async def process_with_tools(user_input: str) -> str:
    """Process a message with tool calling."""
    model = create_model(tools=get_tools())
    
    messages = [HumanMessage(content=user_input)]
    
    logger.debug(f"User input: {user_input}")
    
    while True:
        response = model.invoke(messages)
        logger.debug(f"Response type: {type(response)}")
        logger.debug(f"Response: {response}")
        
        if hasattr(response, "tool_calls") and response.tool_calls:
            for tool_call in response.tool_calls:
                tool_name = tool_call["name"]
                tool_args = tool_call["args"]
                
                logger.info(f"Calling tool: {tool_name} with args: {tool_args}")
                
                tool = next((t for t in get_tools() if t.name == tool_name), None)
                
                if tool:
                    result = tool.invoke(tool_args)
                    logger.info(f"Tool result: {result[:200]}...")
                else:
                    result = f"Error: Unknown tool '{tool_name}'"
                
                messages.append(
                    ToolMessage(content=result, tool_call_id=tool_call["id"])
                )
        else:
            return response.content
```

## Key Takeaways

- **bind_tools()** attaches tools to the model
- **tool_calls** attribute indicates model wants to call a tool
- **ToolMessage** returns tool results to the model
- **Tool names and descriptions** help the LLM decide when to call tools
- **invoke()** executes tools based on the schema

## Complete Tool Set

Your tools module should look like:

```python
"""Tools for the Deep Agents CLI."""

from deepagents_cli.tools.read_file import ReadFileTool, read_file_tool
from deepagents_cli.tools.write_file import write_file
from deepagents_cli.tools.execute import execute

__all__ = [
    "ReadFileTool",
    "read_file_tool", 
    "write_file",
    "execute",
]
```

## Next Section

[Quiz](./quiz.md) — Test your understanding of the Tool System.
