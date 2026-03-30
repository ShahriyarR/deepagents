# Module 5: Middleware Pipeline

Compose reusable behaviors that intercept and modify agent LLM calls.

## Learning Objectives

By the end of this module, you will:

- Understand the middleware interceptor pattern
- Learn how `wrap_model_call()` modifies requests and responses
- Build middleware that injects context into system prompts
- Compose multiple middleware pieces into a pipeline
- Create a `ConfigurableModelMiddleware` for runtime model switching

## Prerequisites

- Module 4 completed (Agent Architecture)
- Understanding of Python classes and inheritance
- Familiarity with async/await patterns
- Basic understanding of LangGraph state

## Estimated Time

~3-4 hours

## Sections

1. [What is Middleware?](./section-01-what-is-middleware.md) — Interceptor pattern explained
2. [Build the Base Class](./section-02-build-base-class.md) — AgentMiddleware foundation
3. [Implement wrap_model_call()](./section-03-implement-wrap.md) — Request/response interception
4. [Compose Middleware](./section-04-compose-middleware.md) — Build the middleware stack
5. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, you'll have a middleware system that:

```python
# Middleware intercepts every LLM call
agent_middleware = [
    ConfigurableModelMiddleware(),   # Swap models at runtime
    MemoryMiddleware(sources=[...]),  # Inject AGENTS.md context
    SkillsMiddleware(sources=[...]), # Load skill documentation
]

# Each middleware wraps the call in a chain:
# Request → Middleware1 → Middleware2 → ... → LLM
# Response ← Middleware1 ← Middleware2 ← ... ← LLM
```

## Key Concepts

- **Interceptor pattern** — Middleware intercepts requests before they reach the LLM
- **wrap_model_call()** — Synchronous hook for modifying requests/responses
- **awrap_model_call()** — Async hook for async operations
- **Middleware composition** — Ordering matters; last registered runs first
- **Request modification** — Add system prompt content, change model settings

## Architecture Preview

```
┌─────────────────────────────────────────────────────────────┐
│                      Your Code                               │
│                                                              │
│  agent = create_deep_agent(middleware=[                     │
│      ConfigurableModelMiddleware(),                         │
│      MemoryMiddleware(),                                     │
│      SkillsMiddleware(),                                     │
│  ])                                                         │
│                                                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                   Middleware Pipeline                        │
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  Config     │    │   Memory    │    │   Skills    │     │
│  │  Model      │───▶│  Middleware │───▶│  Middleware │     │
│  │  Middleware │    │             │    │             │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│        │                  │                  │               │
│        ▼                  ▼                  ▼               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    LLM Call                         │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

## Real-World Examples

Middleware enables powerful behaviors:

- **Inject project context** from `AGENTS.md` files
- **Load skills documentation** from a skills directory
- **Switch models** per-invocation without recompiling
- **Add MCP server info** to system prompts
- **Implement human-in-the-loop** approval flows

## Next Module

[Module 6: Filesystem Tools](../module-06-filesystem-tools/README.md) — Build file operations (ls, read, write, edit).
