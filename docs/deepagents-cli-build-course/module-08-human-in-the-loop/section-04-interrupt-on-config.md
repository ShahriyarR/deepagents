# Section 4: Configuring interrupt_on

Fine-tune which tools require approval.

## The interrupt_on Parameter

LangGraph tools accept an `interrupt_on` parameter that controls when they pause:

```python
tool_config = {
    "interrupt_on": {
        "execute": {
            "allowed_decisions": ["approve", "reject"],
            "description": "Run a shell command",
        },
        "write_file": {
            "allowed_decisions": ["approve", "reject"],
            "description": "Write content to a file",
        },
    }
}
```

When a tool has `interrupt_on` configured and the value is truthy, the tool triggers an interrupt before execution.

## Tool Configuration Options

Each tool can have one of three `interrupt_on` values:

| Value | Behavior |
|-------|----------|
| `True` | Always interrupt for this tool |
| `False` | Never interrupt for this tool |
| `InterruptOnConfig` | Interrupt with specific configuration |

```python
# Always pause for execute
"interrupt_on": {
    "execute": True,
}

# Never pause for read operations  
"interrupt_on": {
    "read_file": False,
}

# Pause with custom config
"interrupt_on": {
    "execute": {
        "allowed_decisions": ["approve", "reject"],
        "description": "Run shell command with arguments shown",
    },
}
```

## Full Configuration in _add_interrupt_on

The `_add_interrupt_on()` function returns the complete configuration:

```python
def _add_interrupt_on() -> dict[str, InterruptOnConfig]:
    """Configure human-in-the-loop interrupt settings for all gated tools."""
    
    return {
        "execute": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_execute_description,
        },
        "write_file": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_write_file_description,
        },
        "edit_file": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_edit_file_description,
        },
        "web_search": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_web_search_description,
        },
        "fetch_url": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_fetch_url_description,
        },
        "task": {
            "allowed_decisions": ["approve", "reject"],
            "description": _format_task_description,
        },
        "launch_async_subagent": {
            "allowed_decisions": ["approve", "reject"],
            "description": "Launch a remote async subagent",
        },
        "update_async_subagent": {
            "allowed_decisions": ["approve", "reject"],
            "description": "Update a remote async subagent",
        },
        "cancel_async_subagent": {
            "allowed_decisions": ["approve", "reject"],
            "description": "Cancel a remote async subagent",
        },
    }
```

## Integration with create_cli_agent

Wire it together in `create_cli_agent`:

```python
def create_cli_agent(
    model: str | BaseChatModel,
    assistant_id: str,
    *,
    auto_approve: bool = False,
    # ... other params
):
    # Configure interrupt_on based on auto_approve
    if auto_approve:
        interrupt_on: dict[str, bool | InterruptOnConfig] = {}
    else:
        interrupt_on = _add_interrupt_on()
    
    # Build the graph
    graph = StateGraph(AgentState)
    
    # Add tools with interrupt_on configuration
    # ... tool setup with interrupt_on=interrupt_on
    
    return graph.compile(interrupt_on=interrupt_on)
```

## Description Formatters

Dynamic descriptions for richer prompts:

```python
def _format_execute_description(pending: dict[str, Any]) -> str:
    """Format shell command description."""
    command = pending.get("args", {}).get("command", "")
    if len(command) > 60:
        command = command[:60] + "..."
    return f"Run shell command: {command}"

def _format_write_file_description(pending: dict[str, Any]) -> str:
    """Format write file description."""
    path = pending.get("args", {}).get("path", "")
    return f"Write to file: {path}"

def _format_edit_file_description(pending: dict[str, Any]) -> str:
    """Format edit file description."""
    path = pending.get("args", {}).get("path", "")
    old = pending.get("args", {}).get("old_string", "")[:30]
    return f"Edit {path}: replace '{old}...'"

def _format_task_description(pending: dict[str, Any]) -> str:
    """Format task delegation description."""
    task_type = pending.get("args", {}).get("task_type", "")
    prompt = pending.get("args", {}).get("prompt", "")[:50]
    return f"Delegate to {task_type}: {prompt}..."
```

## Granular Control

For more nuanced control, use callable descriptions:

```python
def _create_interrupt_on(
    session_state: SessionState,
) -> dict[str, InterruptOnConfig]:
    """Create interrupt config based on session state."""
    
    config: dict[str, InterruptOnConfig] = {}
    
    # Always require approval for shell
    config["execute"] = {
        "allowed_decisions": ["approve", "reject"],
        "description": lambda p: f"Execute: {p['args']['command'][:50]}",
    }
    
    # Only require approval outside safe directory
    if not session_state.in_safe_directory:
        config["write_file"] = {
            "allowed_decisions": ["approve", "reject"],
            "description": "Write file outside safe directory",
        }
    
    return config
```

## Key Takeaways

- `interrupt_on` accepts `True`, `False`, or `InterruptOnConfig`
- `_add_interrupt_on()` centralizes configuration for all tools
- Description can be static strings or callables for dynamic content
- Wire interrupt_on into `create_cli_agent` based on `auto_approve`

## Next Section

[Auto-Approve for CI](./section-05-auto-approve.md) — Batch mode and scripted usage.
