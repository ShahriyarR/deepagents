# Module 3: Tool System

Give the agent capabilities through LangChain tools.

## Learning Objectives

By the end of this module, you will:

- Understand what LangChain tools are and why they matter
- Define tools using the `@tool` decorator
- Create tool arguments using Pydantic `BaseModel`
- Build ReadFileTool, WriteFileTool, and ExecuteTool
- Connect tools to your LangChain model
- Test tool calling with the agent

## Prerequisites

- Module 2 completed (REPL & Streaming)
- Basic Python classes and type hints
- Understanding of async/await

## Estimated Time

~3-4 hours

## Sections

1. [What is a Tool?](./section-01-what-is-tool.md) — Tools vs prompts, tool calling concept
2. [Build ReadFileTool](./section-02-build-read-tool.md) — Your first LangChain tool
3. [Build WriteFileTool](./section-03-build-write-tool.md) — Writing files with validation
4. [Build ExecuteTool](./section-04-build-execute-tool.md) — Shell command execution
5. [Wire Tools to Model](./section-05-wire-tools.md) — Connect tools to the agent
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, you'll have an agent that can:

```python
>>> Read the file at src/agent.py
The file contains 142 lines of Python code...

>>> Write "Hello, World!" to hello.txt
File written successfully.

>>> Run "ls -la"
total 48
drwxr-xr-x 19 user  user  4096 Mar 27 10:52 .
drwxr-xr-x  8 user  user  4096 Mar 27 10:52 ..
...
```

## Key Concepts

- **@tool decorator** — Marks functions as LangChain tools
- **BaseModel** — Pydantic model for tool arguments
- **Tool function signatures** — How tools expose parameters to the LLM
- **bind_tools()** — Attaching tools to a chat model

## Next Module

[Module 4: Agent Architecture](../module-04-agent-architecture/README.md) — Build the LangGraph state machine.
