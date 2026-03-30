# Section 5: Async Subagents

Remote LangGraph servers and background task management.

## Why Async Subagents?

Synchronous subagents block until completion. For long-running tasks:

- **Blocking** — Main agent waits for result
- **Timeout risk** — Very long tasks may timeout
- **No progress** — Can't monitor or update running tasks

Async subagents solve these by running on remote LangGraph servers:

- **Non-blocking** — Returns immediately with task_id
- **Long-running** — Designed for minutes or hours of work
- **Monitorable** — Check status and update running tasks
- **Concurrent** — Launch many tasks simultaneously

## AsyncSubAgent Specification

```python
from deepagents.middleware.async_subagents import AsyncSubAgent

async_subagent: AsyncSubAgent = {
    "name": "long-running-analysis",
    "description": "Long-running analysis on complex datasets",
    "graph_id": "analysis_agent",  # LangGraph deployment name
    "url": "https://my-deployment.langsmith.dev",
    "headers": {"x-custom-header": "value"},
}
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | str | Unique identifier for the async subagent |
| `description` | str | What this subagent does |
| `graph_id` | str | LangGraph deployment/assistant ID |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `url` | str | Server URL. Omit for ASGI (local) transport |
| `headers` | dict | Additional headers for requests |

## AsyncSubAgentMiddleware

```python
from deepagents.middleware.async_subagents import (
    AsyncSubAgentMiddleware,
    AsyncSubAgent,
)

async_subagents: list[AsyncSubAgent] = [
    {
        "name": "long-running-analysis",
        "description": "Long-running analysis agent",
        "graph_id": "analysis_agent",
        "url": "https://my-deployment.langsmith.dev",
    }
]

middleware = AsyncSubAgentMiddleware(
    async_subagents=async_subagents,
)
```

The middleware adds five tools to the agent:

| Tool | Purpose |
|------|---------|
| `start_async_task` | Launch a background task |
| `check_async_task` | Get status and result |
| `update_async_task` | Send follow-up to running task |
| `cancel_async_task` | Stop a running task |
| `list_async_tasks` | List all tracked tasks |

## AsyncTask State

Tasks are tracked in agent state under `async_tasks`:

```python
from deepagents.middleware.async_subagents import AsyncTask

task: AsyncTask = {
    "task_id": "thread-123",
    "agent_name": "long-running-analysis",
    "thread_id": "thread-123",
    "run_id": "run-456",
    "status": "running",  # running, success, error, cancelled
    "created_at": "2024-01-15T10:30:00Z",
    "last_checked_at": "2024-01-15T10:35:00Z",
    "last_updated_at": "2024-01-15T10:30:00Z",
}
```

## Tool Usage Patterns

### Starting a Task

```python
start_async_task(
    description="Analyze the entire codebase for security vulnerabilities...",
    subagent_type="long-running-analysis",
)
# Returns: "Launched async subagent. task_id: thread-123"
```

The agent should report the task_id to the user and return control immediately.

### Checking Task Status

```python
check_async_task(task_id="thread-123")
# Returns: {"status": "running"} or {"status": "success", "result": "..."}
```

### Updating a Running Task

```python
update_async_task(
    task_id="thread-123",
    message="Also check for performance issues...",
)
```

### Cancelling a Task

```python
cancel_async_task(task_id="thread-123")
```

### Listing All Tasks

```python
list_async_tasks(status_filter="running")
# Returns status of all tracked tasks
```

## Workflow Example

```
Main Agent: start_async_task(description="Analyze codebase...", subagent_type="analysis")
           ↓
Server: Creates thread, starts run, returns task_id
           ↓
Main Agent: "Started analysis task. task_id: thread-123"
           ↓
User: (continues other work while task runs)
           ↓
User: "Check the analysis task"
           ↓
Main Agent: check_async_task(task_id="thread-123")
           ↓
Server: Returns current status
           ↓
Main Agent: "The analysis is still running" (or "completed with result: ...")
```

## Authentication

Auth is handled via environment variables:

- `LANGGRAPH_API_KEY`
- `LANGSMITH_API_KEY`
- `LANGCHAIN_API_KEY`

The LangGraph SDK reads these automatically.

## Client Caching

The middleware caches SDK clients for efficiency:

```python
class _ClientCache:
    def __init__(self, agents: dict[str, AsyncSubAgent]) -> None:
        self._agents = agents
        self._sync: dict[tuple, SyncLangGraphClient] = {}
        self._async: dict[tuple, LangGraphClient] = {}
    
    def _cache_key(self, spec: AsyncSubAgent) -> tuple:
        return (spec.get("url"), frozenset(_resolve_headers(spec).items()))
```

Clients are reused across tool invocations for the same agent type.

## State Schema

AsyncSubAgentMiddleware extends agent state:

```python
class AsyncSubAgentState(AgentState):
    async_tasks: Annotated[NotRequired[dict[str, AsyncTask]], _tasks_reducer]
```

The `_tasks_reducer` merges updates into the existing tasks dict.

## Critical Usage Rules

The system prompt enforces strict rules:

1. **Always return control immediately** — Never auto-check after launching
2. **Never poll in a loop** — Check once per user request only
3. **Report stale statuses** — Conversation history statuses are always stale
4. **Show full task_id** — Never truncate or abbreviate

```
### Critical rules:
- After launching, ALWAYS return control to the user immediately.
- Never auto-check after launching.
- Never poll `check_async_task` in a loop.
- If a check returns "running", tell the user and wait.
- Task statuses in conversation history are ALWAYS stale.
```

## Comparison: Sync vs Async

| Aspect | Sync SubAgent | Async SubAgent |
|--------|--------------|----------------|
| **Pattern** | Blocking | Non-blocking |
| **Returns** | Result directly | task_id immediately |
| **Execution** | In-process | Remote server |
| **Monitoring** | Not possible | Full status tracking |
| **Updates** | Not possible | Can update running |
| **Use case** | Quick parallel tasks | Long-running tasks |

## Integration with SubAgentMiddleware

Both middlewares can be used together:

```python
agent = create_deep_agent(
    model="openai:gpt-4o",
    middleware=[
        SubAgentMiddleware(
            backend=my_backend,
            subagents=[...],  # Quick sync subagents
        ),
        AsyncSubAgentMiddleware(
            async_subagents=[...],  # Long-running async
        ),
    ],
)
```

The main agent decides which to use based on task characteristics.

## Error Handling

Errors are returned as strings, not raised:

```python
def start_async_task(description, subagent_type, runtime):
    try:
        client = clients.get_sync(subagent_type)
        # ... create thread and run
    except Exception as e:
        logger.warning("Failed to launch async subagent: %s", e)
        return f"Failed to launch async subagent: {e}"
```

This prevents tool errors from crashing the agent.

## Key Takeaways

- **AsyncSubAgent** — TypedDict for remote LangGraph deployments
- **Non-blocking** — Returns task_id immediately
- **Five tools** — start, check, update, cancel, list
- **State tracking** — Tasks persisted in agent state
- **Critical rules** — Never poll, always return control immediately
- **Client caching** — Efficient SDK client reuse

## Next Section

[Quiz](./quiz.md) — Test your understanding of subagents.

(End of file - 148 lines)
