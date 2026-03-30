# Deep Agents CLI Build Course

Build a production-quality AI coding CLI from scratch.

## Overview

This comprehensive 18-module course teaches you how to build the Deep Agents CLI — an interactive AI coding assistant with:

- **Streaming responses** — See AI output as it's generated
- **Tool calling** — File operations, shell commands, search
- **Human-in-the-loop** — Approval gates for dangerous operations
- **Conversation history** — Context persists across sessions
- **Skills system** — Load custom instructions from markdown files
- **Memory system** — Persistent context via AGENTS.md
- **MCP integration** — Connect to Model Context Protocol servers
- **Subagents** — Spawn parallel agents for complex tasks
- **Remote sandboxes** — Run code in cloud VMs
- **Polished TUI** — Beautiful terminal interface with Textual

## Course Structure

| Module | Topic | Estimated Time |
|--------|-------|---------------|
| 1 | [Project Setup](../deepagents-cli-build-course/module-01-project-setup/README.md) | 2-3 hours |
| 2 | [REPL & Streaming](../deepagents-cli-build-course/module-02-repl-streaming/README.md) | 2-3 hours |
| 3 | [Tool System](../deepagents-cli-build-course/module-03-tool-system/README.md) | 2-3 hours |
| 4 | [Agent Architecture](../deepagents-cli-build-course/module-04-agent-architecture/README.md) | 3-4 hours |
| 5 | [Middleware Pipeline](../deepagents-cli-build-course/module-05-middleware-pipeline/README.md) | 2-3 hours |
| 6 | [Filesystem Tools](../deepagents-cli-build-course/module-06-filesystem-tools/README.md) | 3-4 hours |
| 7 | [Shell Execution](../deepagents-cli-build-course/module-07-shell-execution/README.md) | 2-3 hours |
| 8 | [Human-in-the-Loop](../deepagents-cli-build-course/module-08-human-in-the-loop/README.md) | 2-3 hours |
| 9 | [Session & History](../deepagents-cli-build-course/module-09-session-history/README.md) | 3-4 hours |
| 10 | [Skills System](../deepagents-cli-build-course/module-10-skills-system/README.md) | 2-3 hours |
| 11 | [Memory System](../deepagents-cli-build-course/module-11-memory-system/README.md) | 2-3 hours |
| 12 | [Backend Abstraction](../deepagents-cli-build-course/module-12-backend-abstraction/README.md) | 2-3 hours |
| 13 | [MCP Integration](../deepagents-cli-build-course/module-13-mcp-integration/README.md) | 3-4 hours |
| 14 | [Subagents](../deepagents-cli-build-course/module-14-subagents/README.md) | 3-4 hours |
| 15 | [Remote Sandboxes](../deepagents-cli-build-course/module-15-remote-sandboxes/README.md) | 2-3 hours |
| 16 | [Textual TUI](../deepagents-cli-build-course/module-16-textual-tui/README.md) | 3-4 hours |
| 17 | [Slash Commands](../deepagents-cli-build-course/module-17-slash-commands/README.md) | 2-3 hours |
| 18 | [Polish & Production](../deepagents-cli-build-course/module-18-polish-production/README.md) | 3-4 hours |

**Total estimated time:** 40-60 hours

## What You'll Build

By the end of this course, you'll have a working CLI called `deepagents`:

```bash
# Start interactive session
deepagents run

# List available agents
deepagents list

# Run with specific model
deepagents run --model claude-sonnet-4-20250514

# Auto-approve dangerous operations (CI only!)
deepagents run --auto-approve

# Use remote sandbox
deepagents run --sandbox modal
```

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    CLI Entry Point                     │
│                   main.py (argparse)                   │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│                    Textual TUI                         │
│              app.py (App, Widgets, CSS)                │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│                    LangGraph Agent                     │
│             agent.py (StateGraph, Nodes)                │
└─────────────────────────────────────────────────────┘
                          │
                ┌─────────┴─────────┐
                ▼                   ▼
        ┌───────────────┐   ┌───────────────┐
        │   Middleware   │   │     Tools      │
        │   Pipeline     │   │ (files, shell) │
        └───────────────┘   └───────────────┘
                │                   │
                └─────────┬─────────┘
                          ▼
┌─────────────────────────────────────────────────────┐
│                    LLM Provider                       │
│           (Anthropic Claude via LangChain)              │
└─────────────────────────────────────────────────────┘
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

## Next Steps

1. [Prerequisites](./prerequisites.md) — Check requirements
2. [Quick Start](./quickstart.md) — Set up your environment
3. [Module 1: Project Setup](../deepagents-cli-build-course/module-01-project-setup/README.md) — Start building
