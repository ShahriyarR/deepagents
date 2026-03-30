# Module 1: Project Setup

Build the foundation — a working CLI package with dependencies and entry point.

## Learning Objectives

By the end of this module, you will:

- Initialize a Python project using uv
- Configure pyproject.toml with proper dependencies
- Install dependencies (LangChain, LangGraph, Textual, Anthropic)
- Build a CLI entry point with argparse subcommands
- Verify the CLI works (`deepagents --version`)

## Prerequisites

- Python 3.11+ installed
- uv package manager installed (`pip install uv`)
- Basic terminal familiarity

## Estimated Time

~2-3 hours

## Sections

1. [Architecture Preview](./section-01-architecture.md) — What we're building
2. [Initialize Project](./section-02-initialize-project.md) — `uv init` and package structure
3. [Configure Dependencies](./section-03-dependencies.md) — pyproject.toml and installs
4. [Build Entry Point](./section-04-entry-point.md) — main.py with argparse
5. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, you'll have a working CLI that produces:

```bash
$ deepagents --version
deepagents-cli 0.1.0

$ deepagents --help
usage: deepagents [-h] [--version] {list,run,reset} ...

Deep Agents CLI - Interactive AI coding assistant

options:
  -h, --help            show this help message and exit
  --version             show program version

subcommands:
  {list,run,reset}
    list                List available agents
    run                 Start interactive session
    reset               Reset agent state
```

## Next Module

[Module 2: REPL & Streaming](../module-02-repl-streaming/README.md) — Build the async chat loop with streaming responses.
