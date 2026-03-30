# Module 9: Session & History

Add session management and message history to your CLI.

## Learning Objectives

By the end of this module, you will:

- Understand stateless vs stateful agent architectures
- Generate and manage thread IDs using UUIDs
- Implement SQLite persistence with aiosqlite
- Store and retrieve message history
- Prepend conversation history to prompts
- Use LangGraph checkpointing for state persistence

## Prerequisites

- Completed Module 8: Human-in-the-Loop
- Understanding of Python async/await
- Basic SQL knowledge (INSERT, SELECT)
- Familiarity with LangGraph state management

## Estimated Time

~3-4 hours

## Sections

1. [Stateless vs Stateful](./section-01-stateless-vs-stateful.md) — Understanding session concepts
2. [Generate Thread ID](./section-02-generate-thread-id.md) — UUID generation and management
3. [SQLite Persistence](./section-03-sqlite-persistence.md) — Storing messages with aiosqlite
4. [Prepend History](./section-04-prepend-history.md) — Injecting history into prompts
5. [Checkpointing](./section-05-checkpointing.md) — LangGraph checkpointing for state
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, your CLI will remember conversations:

```python
# First session
>>> Hello, my name is Alice
Hi Alice! Nice to meet you.

# Second session (new process)
>>> Remember me?
Yes Alice, I remember you from our previous conversation!

# Session by ID
>>> session --thread abc123
>>> I told you my name was Bob earlier
Nice to see you again, Bob.
```

## Key Concepts

| Concept | Purpose |
|---------|---------|
| Thread ID | Unique identifier for a conversation session |
| UUID v4 | Random unique identifiers for threads |
| SQLite | Lightweight async database for persistence |
| aiosqlite | Async SQLite driver |
| Message history | Stored conversation turns |
| Prepending | Injecting history into system prompts |
| Checkpointing | LangGraph's built-in state persistence |

## Why Session Management?

Without sessions, every conversation starts fresh:

```
Stateless: "Who are you?"
AI: "I don't know who you are. How can I help?"

Stateful: "Who are you?"
AI: "You're Alice, and you work on the CLI project."
```

Sessions enable:

- **Continuity** — Remember context across messages
- **Multi-turn tasks** — Complex workflows that span multiple exchanges
- **User profiles** — Remember user preferences and names
- **Debugging** — Replay sessions to investigate issues

## Next Module

[Module 10: Skills System](../module-10-skills-system/README.md) — Load custom behaviors from SKILL.md files.
