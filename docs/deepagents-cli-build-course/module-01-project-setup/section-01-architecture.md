# Section 1: Architecture Preview

Before writing code, understand what we're building.

## What is the Deep Agents CLI?

The Deep Agents CLI is an **interactive AI coding assistant** that runs in your terminal. Unlike simple chatbots, it can:

- **Read and write files** — Edit code directly
- **Execute shell commands** — Run tests, git, build tools
- **Use tools** — Search files, browse directories
- **Remember conversations** — Context persists across sessions
- **Ask for approval** — Pauses before dangerous operations
- **Load custom skills** — Extend behavior with markdown files

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        CLI Entry Point                        │
│                     main.py (argparse)                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Textual TUI App                          │
│              app.py (App, Widgets, CSS)                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       LangGraph Agent                         │
│              agent.py (StateGraph, Nodes)                     │
└─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
            ┌───────────────┐   ┌───────────────┐
            │   Middleware   │   │     Tools      │
            │ Pipeline       │   │ (files, shell) │
            └───────────────┘   └───────────────┘
                    │                   │
                    └─────────┬─────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       LLM Provider                           │
│              (Anthropic Claude via LangChain)                 │
└─────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility |
|-----------|----------------|
| `main.py` | CLI argument parsing, entry point dispatch |
| `app.py` | Textual UI, widget composition, event handling |
| `agent.py` | LangGraph state machine, node definitions |
| `tools/*.py` | Tool definitions (ReadFile, WriteFile, Execute) |
| `middleware/*.py` | Request/response interception, modification |
| `skills/` | SKILL.md loading, injection |
| `memory/` | AGENTS.md loading, context injection |

## Data Flow

1. User types message in Textual TUI
2. Message sent to LangGraph agent
3. Agent decides: use tool or respond
4. If tool: middleware intercepts, can modify/approve
5. Tool executes (filesystem, shell, etc.)
6. Result returns to agent
7. Agent generates response
8. Response streams back to TUI
9. TUI displays streaming tokens

## Key Technologies

| Technology | Purpose | Why |
|------------|---------|-----|
| **uv** | Package manager | Fast, modern, built-in virtualenvs |
| **LangChain** | LLM abstractions | Unified interface for LLMs |
| **LangGraph** | Agent framework | Graph-based state machine |
| **Anthropic** | LLM provider | Claude models |
| **Textual** | TUI framework | Async-native, reactive widgets |

## Project Structure

```
deepagents/
├── pyproject.toml          # Package config
├── src/
│   └── deepagents_cli/
│       ├── __init__.py
│       ├── __main__.py    # Entry point
│       ├── main.py        # CLI logic
│       └── ...
└── .venv/                 # Virtual environment
```

## Key Concepts Introduced in This Module

- **uv** — Fast Python package manager with built-in venv support
- **pyproject.toml** — Modern Python package configuration
- **argparse** — CLI argument parsing with subcommands
- **entry points** — Console scripts defined in pyproject.toml

## Next Section

[Initialize Project](./section-02-initialize-project.md) — Create the package with uv.
