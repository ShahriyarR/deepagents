# Section 1: Why Subagents?

Understanding the divide-and-conquer pattern for agent delegation.

## The Problem

Complex tasks overwhelm a single agent:

- **Context overflow** — Large codebases, many files, extensive research
- **Token limits** — Hitting context windows on long conversations
- **Parallel work** — Independent tasks must wait for sequential completion
- **Focus fragmentation** — Switching between domains dilutes expertise

A single agent trying to do everything becomes slow, expensive, and error-prone.

## The Solution: Divide and Conquer

Subagents apply the divide-and-conquer principle:

1. **Split** — Break complex tasks into independent subtasks
2. **Delegate** — Assign each subtask to a specialized subagent
3. **Execute** — Run subagents in parallel (when independent)
4. **Synthesize** — Combine results into a coherent response

```
┌─────────────────────────────────────────────────────┐
│                    Main Agent                       │
│                                                      │
│   User: "Research AI safety, write tests, update     │
│          docs, and deploy to production"            │
│                                                      │
│         ┌─────────────┬─────────────┬────────────┐  │
│         ▼             ▼             ▼            │  │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐        │  │
│   │Researcher│ │ Tester   │ │ Docs     │  ...   │  │
│   │ Subagent │ │ Subagent │ │ Subagent │        │  │
│   └──────────┘ └──────────┘ └──────────┘        │  │
│         │             │             │            │  │
│         └─────────────┴─────────────┘            │  │
│                       │                          │  │
│                       ▼                          │  │
│              ┌─────────────────┐                │  │
│              │ Synthesize +    │                │  │
│              │ User Response   │                │  │
│              └─────────────────┘                │  │
└─────────────────────────────────────────────────────┘
```

## Benefits of Subagents

### Context Isolation

Each subagent operates in its own context window:

- Main agent maintains conversation history
- Subagent receives fresh context for its task
- No context pollution between subtasks
- Token usage is contained per subagent

### Parallel Execution

Independent subagents run simultaneously:

```
Sequential (no subagents):
  Task A: 10s → Task B: 10s → Task C: 10s = 30s total

Parallel (with subagents):
  Task A: 10s
  Task B: 10s  }  →  10s total
  Task C: 10s
```

### Specialized Expertise

Subagents can be tailored for specific domains:

```python
subagents = [
    {
        "name": "security-reviewer",
        "description": "Focused security analysis",
        "system_prompt": "You are a security expert. Analyze for...",
        "tools": [grep_tool, read_tool],  # Narrow toolset
    },
    {
        "name": "performance-analyst",
        "description": "Performance profiling specialist",
        "system_prompt": "You are a performance expert. Analyze for...",
        "tools": [grep_tool, read_tool, profile_tool],
    },
]
```

### Clean Results

Subagents return structured, synthesized output:

- Main agent asks for specific output format
- Subagent does deep work, returns concise result
- Main agent synthesizes without intermediate noise
- User sees clean, aggregated response

## When to Use Subagents

Use the `task` tool when:

- Task is complex and multi-step
- Task can be fully delegated in isolation
- Task is independent of other tasks (can run in parallel)
- Task requires focused reasoning or heavy token usage
- You only care about the final output, not steps

Do NOT use subagents when:

- Task is trivial (few tool calls, simple lookup)
- Delegating adds latency without benefit
- You need to see intermediate reasoning
- Tasks are dependent (must run sequentially)

## Subagent Types

The SDK supports two subagent patterns:

| Type | Pattern | Use Case |
|------|---------|----------|
| **Sync** | Blocks until complete, returns result | Quick parallel tasks, isolated context |
| **Async** | Returns immediately with task_id | Long-running tasks, remote servers |

Both types share the same goal: delegation with clean results.

## Architecture Overview

```
                    ┌─────────────────────────────────────┐
                    │           SubAgentMiddleware         │
                    │                                      │
                    │  ┌────────────────────────────────┐  │
                    │  │      task tool                │  │
                    │  │  (spawns sync subagents)      │  │
                    │  └────────────────────────────────┘  │
                    └─────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    │                                   │
              ┌──────────┐                       ┌──────────────┐
              │ Subagent │                       │   Async      │
              │  (sync)  │                       │  SubAgent    │
              │          │                       │  Middleware  │
              └──────────┘                       └──────────────┘
                                                         │
                                          ┌──────────────┼──────────────┐
                                          │              │              │
                                    start_async    check_async    update_async
                                          │              │              │
                                          └──────────────┴──────────────┘
                                                        │
                                          ┌─────────────┴─────────────┐
                                          │  Remote LangGraph Server   │
                                          └────────────────────────────┘
```

## Key Takeaways

- **Divide and conquer** — Split complex tasks into independent subtasks
- **Context isolation** — Each subagent has its own context window
- **Parallel execution** — Independent subagents run simultaneously
- **Specialized expertise** — Narrow tool sets for focused domains
- **Clean output** — Subagents return synthesized results, not raw work
- **Two patterns** — Sync (blocking) and async (background) subagents

## Next Section

[SubAgent Specification](./section-02-subagent-spec.md) — Learn the TypedDict format for defining subagents.

(End of file - total 131 lines)
