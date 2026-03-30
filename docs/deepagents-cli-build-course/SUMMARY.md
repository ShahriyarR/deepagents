# Deep Agents CLI Build Course

Build a production-quality AI coding CLI from scratch.

## Overview

This comprehensive 18-module course teaches you how to build the Deep Agents CLI — an interactive AI coding assistant with streaming responses, tool calling, human-in-the-loop approval, conversation history, skills, subagents, MCP integration, sandboxes, and a polished Textual TUI.

By the end, you'll have a working agentic coding CLI that you can extend and customize.

## Prerequisites

- Comfortable with Python (functions, classes, imports, async/await)
- Basic understanding of terminal usage
- No prior LangChain or Textual experience required

See [Prerequisites](../course/prerequisites.md) for details.

## Quick Start

1. [Set up your environment](../course/quickstart.md)
2. Start with [Module 1: Project Setup](module-01-project-setup/README.md)

## Course Modules

| Module | Title | Description |
|--------|-------|-------------|
| 01 | [Project Setup](module-01-project-setup/README.md) | Package scaffold, dependencies, CLI entry point |
| 02 | [REPL & Streaming](module-02-repl-streaming/README.md) | Async chat loop, streaming responses |
| 03 | [Tool System](module-03-tool-system/README.md) | LangChain tools, @tool decorator |
| 04 | [Agent Architecture](module-04-agent-architecture/README.md) | LangGraph, state, graph construction |
| 05 | [Middleware Pipeline](module-05-middleware-pipeline/README.md) | Middleware composition, wrapping |
| 06 | [Filesystem Tools](module-06-filesystem-tools/README.md) | ls, read, write, edit, glob, grep |
| 07 | [Shell Execution](module-07-shell-execution/README.md) | subprocess, command allowlisting |
| 08 | [Human-in-the-Loop](module-08-human-in-the-loop/README.md) | Interrupt handling, approval flows |
| 09 | [Session & History](module-09-session-history/README.md) | Thread IDs, SQLite persistence |
| 10 | [Skills System](module-10-skills-system/README.md) | SKILL.md loading, SkillsMiddleware |
| 11 | [Memory System](module-11-memory-system/README.md) | AGENTS.md, MemoryMiddleware |
| 12 | [Backend Abstraction](module-12-backend-abstraction/README.md) | LocalBackend, SandboxBackend, protocol |
| 13 | [MCP Integration](module-13-mcp-integration/README.md) | MCP clients, mcp.json config |
| 14 | [Subagents](module-14-subagents/README.md) | Sync/async subagent spawning |
| 15 | [Remote Sandboxes](module-15-remote-sandboxes/README.md) | Modal, Daytona, remote execution |
| 16 | [Textual TUI](module-16-textual-tui/README.md) | App, widgets, CSS styling |
| 17 | [Slash Commands](module-17-slash-commands/README.md) | Command registry, bypass tiers |
| 18 | [Polish & Production](module-18-polish-production/README.md) | Testing, packaging, CI |

## What You'll Build

```
deepagents/
├── deepagents_cli/           # Your CLI package
│   ├── __init__.py
│   ├── main.py              # Entry point
│   ├── app.py               # Textual app
│   ├── agent.py              # LangGraph agent
│   ├── tools/                # Tool definitions
│   ├── middleware/          # Middleware pipeline
│   ├── skills/              # Skills loading
│   └── ui/                  # Textual widgets
└── pyproject.toml           # Package config
```

## Key Technologies

| Technology | Purpose |
|------------|---------|
| **uv** | Fast Python package manager |
| **LangChain** | LLM abstractions and tool system |
| **LangGraph** | Graph-based agent architecture |
| **Anthropic** | Claude models |
| **Textual** | Async-native TUI framework |
| **SQLite** | Conversation persistence |

## Getting Help

- Open an issue at https://github.com/langchain-ai/deepagents/issues
- Check the [Deep Agents documentation](https://docs.langchain.com/oss/python/deepagents/)
