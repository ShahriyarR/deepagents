# Section 3: SubAgentMiddleware

Building the middleware that adds the task tool to the agent.

## What is SubAgentMiddleware?

`SubAgentMiddleware` is LangChain `AgentMiddleware` that:

1. **Validates subagent specs** — Ensures required fields are present
2. **Creates subagent runnables** — Builds agent graphs from specifications
3. **Builds the task tool** — StructuredTool for invoking subagents
4. **Injects system prompt** — Adds delegation instructions to the main agent

```python
from deepagents.middleware.subagents import SubAgentMiddleware

middleware = SubAgentMiddleware(
    backend=my_backend,
    subagents=[
        {"name": "researcher", "description": "...", ...}
    ],
)
```

## Middleware Initialization

```python
class SubAgentMiddleware(AgentMiddleware[Any, ContextT, ResponseT]):
    def __init__(
        self,
        *,
        backend: BackendProtocol | BackendFactory | None = None,
        subagents: Sequence[SubAgent | CompiledSubAgent] | None = None,
        system_prompt: str | None = TASK_SYSTEM_PROMPT,
        task_description: str | None = None,
        **deprecated_kwargs: Unpack[_DeprecatedKwargs],
    ) -> None:
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `backend` | BackendProtocol, BackendFactory, None | Backend for file operations |
| `subagents` | Sequence[SubAgent \| CompiledSubAgent] | List of subagent specifications |
| `system_prompt` | str, None | Instructions for main agent about task tool |
| `task_description` | str, None | Custom task tool description |

### Deprecated Parameters (v0.5.0 removal)

| Parameter | Type | Description |
|-----------|------|-------------|
| `default_model` | Any | Default model for subagents |
| `default_tools` | Sequence | Default tools for subagents |
| `default_middleware` | list | Middleware applied to all subagents |
| `default_interrupt_on` | dict | HITL config for default subagent |
| `general_purpose_agent` | bool | Include built-in general-purpose agent |

## Creating Subagent Runnables

The `_get_subagents` method builds agent runnables from specs:

```python
def _get_subagents(self) -> list[_SubagentSpec]:
    specs: list[_SubagentSpec] = []
    
    for spec in self._subagents:
        if "runnable" in spec:
            # CompiledSubAgent - use as-is
            compiled = cast("CompiledSubAgent", spec)
            specs.append({
                "name": compiled["name"],
                "description": compiled["description"],
                "runnable": compiled["runnable"],
            })
            continue
        
        # SubAgent - validate required fields
        if "model" not in spec:
            msg = f"SubAgent '{spec['name']}' must specify 'model'"
            raise ValueError(msg)
        if "tools" not in spec:
            msg = f"SubAgent '{spec['name']}' must specify 'tools'"
            raise ValueError(msg)
        
        # Resolve model if string
        model = spec["model"]
        if isinstance(model, str):
            model = init_chat_model(model)
        
        # Build middleware stack
        middleware: list[AgentMiddleware] = list(spec.get("middleware", []))
        
        # Add HITL middleware if interrupt_on specified
        interrupt_on = spec.get("interrupt_on")
        if interrupt_on:
            middleware.append(HumanInTheLoopMiddleware(interrupt_on=interrupt_on))
        
        # Create the agent runnable
        specs.append({
            "name": spec["name"],
            "description": spec["description"],
            "runnable": create_agent(
                model,
                system_prompt=spec["system_prompt"],
                tools=spec["tools"],
                middleware=middleware,
                name=spec["name"],
            ),
        })
    
    return specs
```

## System Prompt Injection

The middleware updates the system message to inform the main agent:

```python
def wrap_model_call(
    self,
    request: ModelRequest[ContextT],
    handler: Callable[[ModelRequest[ContextT]], ModelResponse[ResponseT]],
) -> ModelResponse[ResponseT]:
    """Update the system message to include instructions on using subagents."""
    if self.system_prompt is not None:
        new_system_message = append_to_system_message(
            request.system_message, 
            self.system_prompt
        )
        return handler(request.override(system_message=new_system_message))
    return handler(request)
```

The system prompt includes:
- When to use the task tool
- Available subagent types and their descriptions
- Usage notes and examples

## Backend Requirement

The new API requires a backend:

```python
middleware = SubAgentMiddleware(
    backend=FilesystemBackend(root_dir="/path/to/files"),
    subagents=[...],
)
```

The backend is passed to subagents for file operations and tool execution.

## Integration with Agent

Add the middleware to the pipeline:

```python
agent = create_deep_agent(
    model="openai:gpt-4o",
    system_prompt="You are a helpful assistant...",
    tools=[...],
    backend=my_backend,
    middleware=[
        ConfigurableModelMiddleware(),
        MemoryMiddleware(...),
        SkillsMiddleware(...),
        SubAgentMiddleware(  # Add subagent support
            backend=my_backend,
            subagents=[...],
        ),
    ],
)
```

Middleware order matters — `SubAgentMiddleware` should be last (closest to LLM) if it only injects into prompts.

## Validation

The middleware validates:

1. **New API requires subagents** — At least one subagent must be specified
2. **SubAgent requires model** — Each SubAgent must specify a model
3. **SubAgent requires tools** — Each SubAgent must specify tools
4. **CompiledSubAgent requires runnable** — Must return state with `messages` key

```python
# Validation error examples
SubAgentMiddleware(backend=my_backend, subagents=[])
# ValueError: At least one subagent must be specified

SubAgentMiddleware(
    backend=my_backend,
    subagents=[{"name": "test", "description": "..."}]
)
# ValueError: SubAgent 'test' must specify 'model'
```

## Deprecated API Support

The middleware supports the legacy API with deprecation warnings:

```python
# Old API (deprecated)
SubAgentMiddleware(
    default_model="openai:gpt-4o",
    default_tools=[...],
    general_purpose_agent=True,
)

# New API (recommended)
SubAgentMiddleware(
    backend=my_backend,
    subagents=[
        {
            "name": "general-purpose",
            "model": "openai:gpt-4o",
            "tools": [...],
            ...
        }
    ],
)
```

The deprecated API will be removed in version 0.5.0.

## Key Takeaways

- **New API requires backend** — Backend for subagent file operations
- **Middleware validates specs** — Required fields enforced at initialization
- **System prompt injection** — Instructions for using the task tool
- **Deprecation warnings** — Legacy API emits warnings, new API recommended
- **State isolation** — Subagents get fresh context, parent state excluded

## Next Section

[The Task Tool](./section-04-task-tool.md) — How the task tool spawns synchronous subagents.

(End of file - total 146 lines)
