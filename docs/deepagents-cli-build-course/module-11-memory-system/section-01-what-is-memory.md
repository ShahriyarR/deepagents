# Section 1: What is Memory?

Understanding persistent context with AGENTS.md files.

## The Problem: Context Without Memory

Imagine you're working on a team project with conventions:

- Python 3.11+ required
- Tests go in `tests/` directory
- Run tests with `make test`
- API keys live in `.env`, never commit it

Without memory, your agent doesn't know these conventions. Every conversation starts fresh with no project context.

## The Solution: AGENTS.md

`AGENTS.md` is a markdown file in your project root that contains **persistent context**:

```markdown
# Project Context

## Environment
- Python 3.11+
- Node.js 20+ for frontend
- PostgreSQL 15+ for database

## Commands
- `make test` — Run unit tests
- `make lint` — Run linters
- `make dev` — Start development server

## Conventions
- Follow Google-style docstrings
- Use `pytest` for testing
- Environment variables in `.env` (never commit!)

## Key Files
- `src/main.py` — Entry point
- `src/api/` — API handlers
- `tests/` — Test suite
```

## Memory vs Skills: A Comparison

| Aspect | AGENTS.md (Memory) | SKILL.md (Skills) |
|--------|-------------------|-------------------|
| **Purpose** | Project context and conventions | How to perform a task |
| **Content** | Facts, conventions, structure | Procedures, step-by-step |
| **Lifecycle** | Persistent across sessions | Loaded on-demand |
| **Scope** | Project-wide | Feature-specific |
| **When loaded** | Every conversation | When referenced |
| **Example** | "We use `make test`" | "How to add a new API endpoint" |

## Why Both?

```
Memory (AGENTS.md)                    Skills (SKILL.md)
─────────────────                     ─────────────────
"Remember this project uses..."        "To add a feature, do X, Y, Z"
"We follow this convention..."        "Here's how our CI pipeline works"
"This file structure is..."            "Debugging procedure for production"
```

Think of memory as **what the agent should always know** and skills as **how the agent should act** in specific situations.

## Real-World Analogy

| Analogy | Memory | Skills |
|---------|--------|--------|
| **Software docs** | README.md (project overview) | API reference (how to use) |
| **Employee onboarding** | Company policies | Job-specific training |
| **Cooking** | Kitchen layout | Recipe cards |

## When to Use AGENTS.md

Use AGENTS.md for:

- **Project structure** — Key directories and their purposes
- **Conventions** — Coding standards, naming conventions
- **Commands** — How to build, test, deploy
- **Environment** — Required tools, versions, setup
- **Important notes** — Security policies, gotchas
- **Team contact** — Who to ask for what

## When to Use SKILL.md

Use SKILL.md for:

- **Procedures** — Step-by-step task instructions
- **Tool usage** — How to use specific CLI tools
- **Troubleshooting** — Debug procedures
- **Domain knowledge** — Specialized expertise

## The Memory Loading Flow

```
1. User starts CLI: deepagents run

2. CLI scans for AGENTS.md:
   ./AGENTS.md
   ../AGENTS.md
   ~/project/AGENTS.md

3. MemoryMiddleware loads content:
   - Read file
   - Parse markdown
   - Extract sections

4. Context injected into system prompt:
   ## Project Memory
   (AGENTS.md content here)

5. Agent responds with project-aware context
```

## Example: Without vs With Memory

**Without Memory:**

```
User: How do I run tests?
Agent: I'm not sure what test framework you use.
       Could you tell me how to run tests?
```

**With Memory (AGENTS.md loaded):**

```
User: How do I run tests?
Agent: Run tests with: make test

      This project uses pytest. Tests are in the tests/ directory.
      Coverage reports are generated in htmlcov/ after running make test.
```

## Key Takeaways

- **AGENTS.md provides persistent context** that survives across sessions
- **Memory is project-wide** — conventions, structure, commands
- **Skills are on-demand** — procedures, how-to guides
- **Both enhance the agent** but serve different purposes
- **Memory loads automatically** at session start

## Next Section

[FilesystemBackend](./section-02-filesystem-backend.md) — Build an abstraction for reading files from disk.
