# Section 3: SkillsMiddleware

Loading skills into the agent via middleware.

## What is SkillsMiddleware?

`SkillsMiddleware` is LangChain `AgentMiddleware` that:

1. **Discovers skills** from configured backend sources
2. **Loads metadata** before agent execution
3. **Injects skills** into the system prompt via progressive disclosure

```python
from deepagents.middleware.skills import SkillsMiddleware

middleware = SkillsMiddleware(
    backend=my_backend,
    sources=["/skills/user/", "/skills/project/"],
)
```

## Middleware State

SkillsMiddleware uses typed state for skill metadata:

```python
from typing import Annotated, NotRequired
from langchain.agents.middleware.types import AgentState, PrivateStateAttr

class SkillsState(AgentState):
    """State for the skills middleware."""
    
    skills_metadata: NotRequired[Annotated[list[SkillMetadata], PrivateStateAttr]]
    """List of loaded skill metadata. Not propagated to parent agents."""
```

`PrivateStateAttr` marks state as local to this agent (not shared with parent graphs).

## Initialization

```python
class SkillsMiddleware(AgentMiddleware[SkillsState, ContextT, ResponseT]):
    state_schema = SkillsState
    
    def __init__(self, *, backend: BACKEND_TYPES, sources: list[str]) -> None:
        self._backend = backend
        self.sources = sources
        self.system_prompt_template = SKILLS_SYSTEM_PROMPT
```

## Backend Resolution

The backend can be a direct instance or a factory function:

```python
# Direct instance
backend = FilesystemBackend(root_dir="/path/to/skills")
middleware = SkillsMiddleware(backend=backend, sources=[...])

# Factory function (for StateBackend with runtime context)
middleware = SkillsMiddleware(
    backend=lambda rt: StateBackend(rt),
    sources=[...],
)
```

The `_get_backend` method resolves the backend:

```python
def _get_backend(self, state: SkillsState, runtime: Runtime, config: RunnableConfig) -> BackendProtocol:
    """Resolve backend from instance or factory."""
    if callable(self._backend):
        tool_runtime = ToolRuntime(
            state=state,
            context=runtime.context,
            stream_writer=runtime.stream_writer,
            store=runtime.store,
            config=config,
            tool_call_id=None,
        )
        return self._backend(tool_runtime)
    return self._backend
```

## Loading Skills: `before_agent`

Skills are loaded once per session in `before_agent`:

```python
def before_agent(self, state: SkillsState, runtime: Runtime, config: RunnableConfig) -> SkillsStateUpdate | None:
    """Load skills metadata before agent execution."""
    
    # Skip if already loaded (handles checkpointed sessions)
    if "skills_metadata" in state:
        return None
    
    backend = self._get_backend(state, runtime, config)
    all_skills: dict[str, SkillMetadata] = {}
    
    # Load from each source in order
    for source_path in self.sources:
        source_skills = _list_skills(backend, source_path)
        for skill in source_skills:
            all_skills[skill["name"]] = skill
    
    skills = list(all_skills.values())
    return SkillsStateUpdate(skills_metadata=skills)
```

The async version:

```python
async def abefore_agent(self, state: SkillsState, runtime: Runtime, config: RunnableConfig) -> SkillsStateUpdate | None:
    """Load skills metadata before agent execution (async)."""
    
    if "skills_metadata" in state:
        return None
    
    backend = self._get_backend(state, runtime, config)
    all_skills: dict[str, SkillMetadata] = {}
    
    for source_path in self.sources:
        source_skills = await _alist_skills(backend, source_path)
        for skill in source_skills:
            all_skills[skill["name"]] = skill
    
    skills = list(all_skills.values())
    return SkillsStateUpdate(skills_metadata=skills)
```

## Key Design: State-Based Loading

Skills are loaded **once per session** into state:

1. First agent call triggers `before_agent`
2. Skills metadata stored in `state.skills_metadata`
3. Subsequent calls check state and skip loading
4. Checkpointed sessions restore metadata automatically

```python
# Pseudocode for middleware execution order
def invoke_agent(state):
    # 1. before_agent runs ONCE (first call or no checkpoint)
    if "skills_metadata" not in state:
        state = load_skills(state)  # Sets state.skills_metadata
    
    # 2. Agent processes
    result = agent.process(state)
    
    # 3. wrap_model_call injects into prompt
    return result
```

## Backend Factory Pattern

StateBackend requires a factory because each agent needs its own instance:

```python
from deepagents.backends.state import StateBackend

# WRONG - all agents share same StateBackend
middleware = SkillsMiddleware(backend=StateBackend(), sources=[...])

# CORRECT - factory creates per-agent instance
middleware = SkillsMiddleware(
    backend=lambda rt: StateBackend(rt),
    sources=[...],
)
```

## Integration with Agent

SkillsMiddleware is added to the middleware pipeline:

```python
agent = create_deep_agent(
    middleware=[
        ConfigurableModelMiddleware(),   # Runtime model switching
        MemoryMiddleware(...),           # AGENTS.md context
        SkillsMiddleware(               # Skills loading
            backend=FilesystemBackend(root_dir="/path/to/skills"),
            sources=["/skills/user/", "/skills/project/"],
        ),
    ]
)
```

Middleware order matters — `SkillsMiddleware` should be last (closest to LLM) if it only injects into prompts.

## Key Takeaways

- **State-based loading** — Skills loaded once, stored in agent state
- **Backend factory** — Supports both direct instances and runtime factories
- **Skip on checkpoint** — Checkpointed sessions restore skills automatically
- **Source ordering** — Later sources override earlier ones (last one wins)

## Next Section

[Load into System Prompt](./section-04-load-into-prompt.md) — Progressive disclosure and system prompt injection.
