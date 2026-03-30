# Section 2: The Interrupt System

LangGraph interrupts pause execution for external input.

## How LangGraph Interrupts Work

LangGraph's `interrupt()` function halts the graph at a defined point:

```python
from langgraph.types import interrupt

def execute_node(state: AgentState) -> AgentState:
    tool_call = state.pending_tool_call
    
    # This pauses execution and returns control to the caller
    decision = interrupt(
        {
            "tool": tool_call.name,
            "args": tool_call.args,
            "description": f"Execute shell command: {tool_call.args['command']}"
        }
    )
    
    if decision == "approve":
        # Proceed with execution
        return {"executed": True}
    else:
        # Reject - stop this branch
        return {"executed": False, "error": "Rejected by user"}
```

When `interrupt()` is called, the graph saves its state and returns to the caller. The caller can then:

1. Display an approval prompt to the user
2. Wait for their decision
3. Resume the graph with the decision via `Command(resume=decision)`

## The Decision Flow

```
Agent Node
    │
    │ [Tool call: execute("rm -rf /tmp/test")]
    ▼
execute_node()
    │
    │ interrupt({"tool": "execute", "args": {...}})
    ▼
[GRAPH PAUSES - returns to Python caller]
    │
    ▼
Textual Adapter receives interrupt payload
    │
    ▼
Approval Prompt displayed to user
    │
    ├── [Approve] → Command(resume="approve") → Resume graph
    │
    └── [Reject]  → Command(resume="reject")  → Stop/abort
```

## InterruptOnConfig

The `InterruptOnConfig` TypedDict defines what each tool requires:

```python
from typing import TypedDict

class InterruptOnConfig(TypedDict, total=False):
    allowed_decisions: list[str]
    description: str | Callable[[], str]
```

Example configuration:

```python
interrupt_config: dict[str, InterruptOnConfig] = {
    "execute": {
        "allowed_decisions": ["approve", "reject"],
        "description": lambda: f"Run shell command: {pending_cmd}",
    },
    "write_file": {
        "allowed_decisions": ["approve", "reject"],
        "description": "Write content to a file",
    },
    "task": {
        "allowed_decisions": ["approve", "reject"],
        "description": "Delegate work to a subagent",
    },
}
```

## Implementing _add_interrupt_on

In `agent.py`, the `_add_interrupt_on()` function builds the configuration:

```python
def _add_interrupt_on() -> dict[str, InterruptOnConfig]:
    """Configure human-in-the-loop interrupt settings for all gated tools."""
    
    execute_interrupt_config: InterruptOnConfig = {
        "allowed_decisions": ["approve", "reject"],
        "description": _format_execute_description,
    }
    
    write_file_interrupt_config: InterruptOnConfig = {
        "allowed_decisions": ["approve", "reject"],
        "description": _format_write_file_description,
    }
    
    # ... more tools
    
    return {
        "execute": execute_interrupt_config,
        "write_file": write_file_interrupt_config,
        "edit_file": edit_file_interrupt_config,
        "web_search": web_search_interrupt_config,
        "fetch_url": fetch_url_interrupt_config,
        "task": task_interrupt_config,
        "launch_async_subagent": async_subagent_interrupt_config,
        "update_async_subagent": async_subagent_interrupt_config,
        "cancel_async_subagent": async_subagent_interrupt_config,
    }
```

## Resuming with Command

After the user decides, resume the graph:

```python
from langgraph.types import Command

# User approved
resume_command = Command(resume="approve")
graph.invoke(None, stream=True, config={"configurable": {"resume": resume_command}})

# User rejected
reject_command = Command(resume="reject")
graph.invoke(None, stream=True, config={"configurable": {"resume": reject_command}})
```

## auto_approve Mode

When `auto_approve=True`, skip interrupts entirely:

```python
def create_cli_agent(
    model: str | BaseChatModel,
    assistant_id: str,
    *,
    auto_approve: bool = False,
    # ...
):
    if auto_approve:
        # No interrupts - all tools run automatically
        interrupt_on: dict[str, bool | InterruptOnConfig] = {}
    else:
        # Full interrupt system
        interrupt_on = _add_interrupt_on()
    
    graph = StateGraph(AgentState)
    # ... build graph with interrupt_on passed to tools
```

## Key Takeaways

- `interrupt()` pauses LangGraph and returns control to Python
- `InterruptOnConfig` defines what decisions are allowed per tool
- User decisions resume the graph via `Command(resume=...)`
- `auto_approve=True` bypasses all interrupts

## Next Section

[Approval Prompts](./section-03-approval-prompts.md) — Building the UI for user decisions.
