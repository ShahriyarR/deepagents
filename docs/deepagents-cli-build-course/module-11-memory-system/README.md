# Module 11: Memory System

Load persistent context from AGENTS.md files to provide project-aware agent behavior.

## Learning Objectives

By the end of this module, you will:

- Understand the difference between AGENTS.md (memory) and SKILL.md (skills)
- Implement a FilesystemBackend for reading files
- Build MemoryMiddleware to intercept LLM requests
- Load and parse AGENTS.md files from project directories
- Inject memory context into the system prompt

## Prerequisites

- Completion of Module 5: Middleware Pipeline
- Completion of Module 10: Skills System
- Understanding of LangChain message types

## Estimated Time

~2-3 hours

## Sections

1. [What is Memory?](./section-01-what-is-memory.md) — Memory vs skills, persistent context
2. [FilesystemBackend](./section-02-filesystem-backend.md) — File reading abstraction
3. [MemoryMiddleware](./section-03-memory-middleware.md) — AGENTS.md loading middleware
4. [Load AGENTS.md](./section-04-load-agents-md.md) — Parsing and validation
5. [Context Injection](./section-05-context-injection.md) — System prompt modification
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, your CLI will automatically detect and load `AGENTS.md` files:

```bash
$ cd ~/my-project
$ deepagents run

# Agent automatically loads ~/my-project/AGENTS.md
# System prompt now includes:
# """
# ## Project Memory
# - This project uses Python 3.11+
# - Main entry point is src/main.py
# - Run tests with: make test
# """
```

## Key Concepts

| Concept | Purpose |
|---------|---------|
| AGENTS.md | Persistent project context (memory) |
| SKILL.md | Single-purpose skill documentation |
| FilesystemBackend | Abstraction for file operations |
| MemoryMiddleware | Intercepts requests to inject memory |

## How Memory Differs from Skills

```
┌─────────────────────────────────────────────────────────────┐
│                      Your Project                             │
│                                                              │
│  AGENTS.md              SKILL.md files                       │
│  (Memory)               (Skills)                             │
│  ─────────              ─────────                            │
│  • Project context      • How to use a feature              │
│  • Team conventions     • Step-by-step procedures            │
│  • Important files      • Tool compositions                  │
│  • Environment setup    • Domain knowledge                   │
│  • Persistent           • On-demand                          │
└─────────────────────────────────────────────────────────────┘
```

## Next Module

[Module 12: Backend Abstraction](../module-12-backend-abstraction/README.md) — Abstract filesystem operations for local and sandboxed execution.
