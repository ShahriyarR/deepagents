# Module 2: REPL & Streaming

Build a basic chat loop that streams AI responses.

## Learning Objectives

By the end of this module, you will:

- Understand streaming vs blocking LLM responses
- Build an async REPL (Read-Eval-Print Loop)
- Connect to Claude via LangChain
- Stream tokens to the terminal in real-time
- Handle user input and maintain conversation context

## Prerequisites

- Module 1 completed
- Basic understanding of async/await

## Estimated Time

~2-3 hours

## Sections

1. [What is Streaming?](./section-01-what-is-streaming.md) — Streaming vs invoke
2. [Build the REPL Skeleton](./section-02-build-repl-skeleton.md) — Input → Loop → Output
3. [Connect the LLM](./section-03-connect-llm.md) — LangChain + Anthropic
4. [Add Streaming](./section-04-add-streaming.md) — Stream tokens in real-time
5. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, you'll have a working chat loop:

```python
>>> What can you do?
I can help you with coding tasks. I can:
- Read and write files
- Execute shell commands
- Search through code
- And more!

>>> exit
Goodbye!
```

## Key Concepts

- **Async generators** — Yield values over time
- **Streaming** — Receive tokens as they're generated
- **REPL** — Read-Eval-Print Loop pattern
- **LangChain message types** — HumanMessage, AIMessage, SystemMessage

## Next Module

[Module 3: Tool System](../module-03-tool-system/README.md) — Give the agent capabilities through tools.
