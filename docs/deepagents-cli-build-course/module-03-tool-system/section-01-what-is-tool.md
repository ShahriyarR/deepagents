# Section 1: What is a Tool?

Understand LangChain tools and the tool calling paradigm.

## The Problem with Prompts Alone

A vanilla LLM can only generate text. It can't:

- Read files from disk
- Write files to disk
- Execute shell commands
- Search the web
- Access databases

You could ask an LLM to "read a file" but it can't actually do it — it would just describe what it would do.

## Tools Extend the LLM's Capabilities

A **tool** is a function that the LLM can actually call to perform actions:

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   User:     │         │     LLM     │         │    Tool     │
│  "Read      │────────▶│  decides    │────────▶│  executes   │
│   file"     │         │  to call    │         │  (filesystem)
│             │         │  tool       │         │             │
└─────────────┘         └──────┬──────┘         └──────┬──────┘
                               │                        │
                               ▼                        ▼
                        ┌─────────────┐         ┌─────────────┐
                        │   Result    │◀────────│   Output    │
                        │  returned   │         │  (file      │
                        │  to LLM     │         │   contents) │
                        └─────────────┘         └─────────────┘
```

## Tool Calling Flow

1. User sends a message (e.g., "Read the file at src/agent.py")
2. LLM recognizes the intent and decides to call a tool
3. LLM returns a "tool call" with the tool name and arguments
4. Framework executes the tool function
5. Tool result returns to the LLM
6. LLM generates a natural language response using the result

## LangChain Tool Example

Here's a simple tool that gets the current working directory:

```python
from langchain_core.tools import tool


@tool
def get_current_directory() -> str:
    """Get the current working directory.
    
    Returns:
        The current working directory path.
    """
    import os
    return os.getcwd()
```

The `@tool` decorator:
- Wraps the function as a LangChain `BaseTool`
- Extracts the docstring for the LLM to understand when to use it
- Infers argument types from type hints

## Tool with Arguments

Most useful tools need input. Use Pydantic `BaseModel` for arguments:

```python
from langchain_core.tools import tool
from pydantic import BaseModel, Field


class ReadFileArgs(BaseModel):
    """Arguments for ReadFile tool."""
    
    file_path: str = Field(description="Path to the file to read")
    max_lines: int = Field(default=100, description="Maximum number of lines to read")


@tool
def read_file(args: ReadFileArgs) -> str:
    """Read contents of a file.
    
    Args:
        args: Contains file_path and optional max_lines.
    
    Returns:
        File contents as a string.
    """
    with open(args.file_path, "r") as f:
        lines = f.readlines()
    
    if args.max_lines:
        lines = lines[:args.max_lines]
    
    return "".join(lines)
```

The LLM sees the `BaseModel` schema and knows:
- Tool name: `read_file`
- It needs a `file_path` (required)
- It can use `max_lines` (optional, defaults to 100)

## Why Use BaseModel for Arguments?

1. **Explicit schema** — LLM knows exactly what inputs are available
2. **Validation** — Pydantic validates inputs before execution
3. **Documentation** — `Field(description=...)` provides context to the LLM
4. **Type safety** — Your code gets typed arguments

## Comparing: Prompt vs Tool

**Without tools** — LLM describes actions:

```
User: "Read the file at src/main.py"
LLM: "I would read the file at src/main.py by opening it and showing you its contents.
      The file likely contains your main application code..."
```

**With tools** — LLM actually performs actions:

```
User: "Read the file at src/main.py"
LLM: [calls read_file with file_path="src/main.py"]
Tool: Returns file contents...
LLM: "Here's the contents of src/main.py:

```python
def main():
    print('Hello, World!')
...
```
"
```

## Tool Naming Conventions

LangChain tools follow these conventions:

| Pattern | Example | Use Case |
|---------|---------|----------|
| `verb_noun` | `read_file` | Action-focused |
| `noun_verb` | `file_reader` | Noun-focused |
| `get_thing` | `get_current_directory` | Getters |

Choose `verb_noun` for most tools — it clearly communicates the action.

## Key Takeaways

- **Tools extend LLMs** — Give them ability to perform real actions
- **@tool decorator** — Converts a function to a LangChain tool
- **BaseModel** — Defines tool arguments with validation
- **Tool calling** — LLM decides when to call, framework executes
- **Docstrings matter** — LLM uses them to decide when to use a tool

## Next Section

[Build ReadFileTool](./section-02-build-read-tool.md) — Create your first LangChain tool.
