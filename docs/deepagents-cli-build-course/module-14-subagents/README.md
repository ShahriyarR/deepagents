# Module 14: Subagents

Enable the agent to spawn subagents for parallel work.

## Learning Objectives

By the end of this module, you will:

- Understand the divide-and-conquer pattern for agent delegation
- Learn the `SubAgent` TypedDict specification format
- Build `SubAgentMiddleware` to add a `task` tool to the agent
- Understand how the task tool spawns synchronous subagents
- Implement `AsyncSubAgentMiddleware` for remote LangGraph servers
- Use `start_async_task` and `check_async_task` for background work

## Prerequisites

- Module 5 completed (Middleware Pipeline)
- Module 11 completed (Memory System)
- Understanding of Python TypedDict and dataclasses
- Familiarity with LangChain agent creation

## Estimated Time

~4-5 hours

## Sections

1. [Why Subagents?](./section-01-why-subagents.md) — Divide and conquer pattern
2. [SubAgent Specification](./section-02-subagent-spec.md) — TypedDict format and fields
3. [SubAgentMiddleware](./section-03-subagent-middleware.md) — Building the task tool
4. [The Task Tool](./section-04-task-tool.md) — Spawning sync subagents
5. [Async Subagents](./section-05-async-subagents.md) — Remote LangGraph servers
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, you'll have a subagent system that:

```python
from deepagents.middleware.subagents import SubAgentMiddleware
from deepagents.middleware.async_subagents import AsyncSubAgentMiddleware

agent = create_agent(
    "openai:gpt-4o",
    middleware=[
        SubAgentMiddleware(
            backend=my_backend,
            subagents=[
                {
                    "name": "researcher",
                    "description": "Research agent for deep analysis",
                    "system_prompt": "You are a research agent...",
                    "model": "openai:gpt-4o",
                    "tools": [search_tool, browse_tool],
                }
            ],
        ),
        AsyncSubAgentMiddleware(
            async_subagents=[
                {
                    "name": "long-running",
                    "description": "Long-running analysis agent",
                    "graph_id": "analysis_agent",
                    "url": "https://my-deployment.langsmith.dev",
                }
            ],
        ),
    ],
)
```

## Key Concepts

- **SubAgent TypedDict** — Specification for named subagents
- **CompiledSubAgent** — Pre-built agent runnables for custom graphs
- **SubAgentMiddleware** — Adds `task` tool for sync subagent spawning
- **Task tool** — Invokes subagents by name with task description
- **AsyncSubAgent** — Specification for remote LangGraph deployments
- **AsyncSubAgentMiddleware** — Tools for background task management
- **start_async_task** — Launch background task, returns task_id immediately
- **check_async_task** — Get status and result of a running task

## Architecture Preview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Main Agent                                  │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │  SubAgent    │    │  AsyncSub   │    │   Other      │     │
│  │  Middleware  │    │  Agent      │    │   Middleware │     │
│  │              │    │  Middleware │    │              │     │
│  └──────────────┘    └──────────────┘    └──────────────┘     │
│        │                  │                  │                    │
│        ▼                  ▼                  ▼                    │
│  ┌──────────────┐    ┌──────────────┐                           │
│  │  task tool   │    │ start_async │                           │
│  │              │    │ check_async │                           │
│  │  update_async│    │ cancel      │                           │
│  │  list_tasks  │    │             │                           │
│  └──────────────┘    └──────────────┘                           │
│        │                  │                                     │
└────────┼──────────────────┼─────────────────────────────────────┘
         │                  │
         ▼                  ▼
   ┌──────────┐      ┌──────────────┐
   │ Subagent │      │ Remote       │
   │ (sync)   │      │ LangGraph    │
   │          │      │ Server       │
   └──────────┘      └──────────────┘
```

## Real-World Examples

Subagents enable powerful delegation patterns:

- **Parallel research** — Launch multiple researcher subagents for different topics simultaneously
- **Code review** — Spawn a focused reviewer agent for security, performance, style
- **Long-running analysis** — Use async subagents for tasks that take minutes to hours
- **Specialized expertise** — Domain-specific subagents with narrow tool access

## Next Module

[Module 15: Remote Sandboxes](../module-15-remote-sandboxes/README.md) — Execute code in isolated Modal sandboxes.

(End of file - total 98 lines)
