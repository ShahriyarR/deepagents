# Section 4: The Task Tool

How the task tool spawns synchronous subagents.

## Task Tool Overview

The `task` tool is a StructuredTool that invokes subagents by name:

```python
from langchain_core.tools import StructuredTool

task_tool = StructuredTool.from_function(
    name="task",
    func=task,           # Sync invocation
    coroutine=atask,     # Async invocation
    description=description,
)
```

## Tool Signature

```python
def task(
    description: str,
    subagent_type: str,
    runtime: ToolRuntime,
) -> str | Command:
    """Invoke a subagent to handle a task."""
    ...
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `description` | str | Detailed task instructions for the subagent |
| `subagent_type` | str | Which subagent to invoke (from available types) |
| `runtime` | ToolRuntime | Runtime context for tool execution |

## Task Tool Description

The tool description lists available subagents:

```python
TASK_TOOL_DESCRIPTION = """Launch an ephemeral subagent to handle complex, 
multi-step independent tasks with isolated context windows.

Available agent types and the tools they have access to:
{available_agents}

When using the Task tool, you must specify a subagent_type parameter 
to select which agent type to use.

## Usage notes:
1. Launch multiple agents concurrently whenever possible...
2. When the agent is done, it will return a single message back to you...
3. Each agent invocation is stateless...
4. The agent's outputs should generally be trusted...
"""
```

The `{available_agents}` placeholder is replaced with the actual subagent list:

```
- researcher: Research agent for deep analysis
- security-reviewer: Security-focused code review
- general-purpose: General purpose agent for all tasks
```

## Invocation Flow

When the main agent calls the task tool:

```
┌─────────────────────────────────────────────────────────────┐
│                     Main Agent                              │
│                                                              │
│  1. task(description="Research AI safety",                  │
│          subagent_type="researcher")                        │
│                        │                                     │
└────────────────────────┼─────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    task tool                                 │
│                                                              │
│  2. Validate subagent_type exists                            │
│  3. Prepare state (exclude parent state keys)                │
│  4. Invoke subagent with fresh messages                       │
│                        │                                     │
└────────────────────────┼─────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   Subagent (researcher)                      │
│                                                              │
│  5. Execute task with tools                                   │
│  6. Return result as ToolMessage                             │
│                        │                                     │
└────────────────────────┼─────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                     Main Agent                              │
│                                                              │
│  7. Receive result in messages                                │
│  8. Synthesize response for user                             │
└─────────────────────────────────────────────────────────────┘
```

## State Preparation

When invoking a subagent, state is prepared to prevent parent context leakage:

```python
def _validate_and_prepare_state(
    subagent_type: str,
    description: str,
    runtime: ToolRuntime,
) -> tuple[Runnable, dict]:
    # Get the subagent runnable
    subagent = subagent_graphs[subagent_type]
    
    # Filter out parent state keys
    subagent_state = {
        k: v 
        for k, v in runtime.state.items() 
        if k not in _EXCLUDED_STATE_KEYS
    }
    
    # Inject task description as fresh message
    subagent_state["messages"] = [HumanMessage(content=description)]
    
    return subagent, subagent_state
```

The excluded state keys are:
- `messages` — Fresh conversation for subagent
- `todos` — Independent task tracking
- `structured_response` — Subagent provides its own
- `skills_metadata` — Subagent loads its own skills
- `memory_contents` — Subagent loads its own memory

## Result Handling

Subagents return results via `Command` with state updates:

```python
def _return_command_with_state_update(
    result: dict,
    tool_call_id: str,
) -> Command:
    # Extract state updates (excluding problematic keys)
    state_update = {
        k: v 
        for k, v in result.items() 
        if k not in _EXCLUDED_STATE_KEYS
    }
    
    # Get the final message
    message_text = result["messages"][-1].text.rstrip()
    
    return Command(
        update={
            **state_update,
            "messages": [ToolMessage(message_text, tool_call_id=tool_call_id)],
        }
    )
```

The main agent receives a `ToolMessage` with the subagent's final output.

## Error Handling

The task tool handles errors gracefully:

```python
def task(description, subagent_type, runtime):
    # Unknown subagent type
    if subagent_type not in subagent_graphs:
        allowed_types = ", ".join([f"`{k}`" for k in subagent_graphs])
        return f"Cannot invoke subagent {subagent_type}. Available: {allowed_types}"
    
    # Missing tool call ID
    if not runtime.tool_call_id:
        raise ValueError("Tool call ID is required for subagent invocation")
    
    # Subagent execution errors propagate
    ...
```

## Usage Examples

### Basic Invocation

```python
# Main agent prompt includes:
# "Use the task tool to research AI safety for the user query"

# Agent calls:
task(
    description="Research the latest developments in AI safety research. "
               "Focus on alignment techniques, interpretability, and "
               "robustness. Return a structured summary with key findings.",
    subagent_type="researcher",
    runtime=runtime,
)
```

### Parallel Subagents

```python
# Agent calls multiple subagents in parallel for independent tasks
task(description="Research AI safety...", subagent_type="researcher")
task(description="Review code for vulnerabilities...", subagent_type="security-reviewer")
task(description="Analyze performance...", subagent_type="performance-analyst")
```

### Custom Output Format

```python
task(
    description="""Analyze the repository for security vulnerabilities.

Required output format:
{
  "vulnerabilities": [
    {
      "severity": "critical|high|medium|low",
      "type": "SQL Injection",
      "location": "src/db.py:42",
      "description": "...",
      "fix": "..."
    }
  ],
  "summary": "..."
}
""",
    subagent_type="security-reviewer",
)
```

## Tool Description Template

You can provide a custom task tool description:

```python
middleware = SubAgentMiddleware(
    backend=my_backend,
    subagents=[...],
    task_description="""Custom task tool description.

Available agents:
{available_agents}

Custom usage notes:
- Only use for complex, multi-step tasks
- Be specific in the description field
"""
)
```

The `{available_agents}` placeholder is automatically replaced with the actual list.

## System Prompt Fragment

The system prompt added to the main agent includes:

```python
TASK_SYSTEM_PROMPT = """## `task` (subagent spawner)

You have access to a `task` tool to launch short-lived subagents...

When to use the task tool:
- When a task is complex and multi-step...
- When a task is independent of other tasks...
- When a task requires focused reasoning...

When NOT to use the task tool:
- If you need to see the intermediate reasoning...
- If the task is trivial...
- If delegating does not reduce token usage...
"""
```

## Key Takeaways

- **task tool** — StructuredTool for invoking subagents by name
- **State isolation** — Parent state filtered, fresh messages for subagent
- **ToolMessage result** — Subagent output wrapped in ToolMessage
- **Error handling** — Graceful degradation on unknown types or errors
- **Parallel capability** — Main agent can launch multiple subagents concurrently

## Next Section

[Async Subagents](./section-05-async-subagents.md) — Remote LangGraph servers and background task management.

(End of file - total 143 lines)
