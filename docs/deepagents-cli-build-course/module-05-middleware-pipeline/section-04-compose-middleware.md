# Section 4: Compose Middleware

Build the middleware pipeline by stacking multiple middleware pieces.

## Middleware Composition

Individual middleware are useful, but the real power comes from **composing them into a pipeline**:

```python
agent = create_deep_agent(
    middleware=[
        ConfigurableModelMiddleware(),   # 1. Runtime model switching
        MemoryMiddleware(sources=[...]), # 2. Inject AGENTS.md context
        SkillsMiddleware(sources=[...]), # 3. Load skill documentation
    ]
)
```

Each middleware wraps the next, forming a chain. The **last middleware** in the list is closest to the actual LLM call.

## How Composition Works

When you pass `middleware=[A, B, C]`:

```
Request flows:    A → B → C → LLM
Response flows:   LLM → C → B → A
```

The handler passed to each middleware is **the next middleware in the chain**:

```python
# Simplified view of how LangChain invokes middleware:

def call_middleware(middlewares, request):
    async def chain(index):
        if index == len(middlewares):
            return await llm.acall(request)  # Actual LLM call
        else:
            mw = middlewares[index]
            # mw's handler is the rest of the chain
            async def handler(req):
                return await chain(index + 1)
            return await mw.awrap_model_call(request, handler)
    
    return chain(0)
```

## Building the Pipeline

### Step 1: Define Middleware Stack

Create a function that builds your middleware stack:

```python
"""middleware_pipeline.py"""

from typing import Sequence

from langchain.agents.middleware.types import AgentMiddleware

from deepagents.middleware import MemoryMiddleware, SkillsMiddleware
from deepagents_cli.configurable_model import ConfigurableModelMiddleware


def build_middleware_stack(
    enable_memory: bool = True,
    enable_skills: bool = True,
) -> Sequence[AgentMiddleware]:
    """Build the middleware pipeline for the CLI agent.

    Args:
        enable_memory: Whether to enable MemoryMiddleware (AGENTS.md loading)
        enable_skills: Whether to enable SkillsMiddleware (skill docs)

    Returns:
        List of middleware in execution order
    """
    middleware: list[AgentMiddleware] = []

    # ConfigurableModelMiddleware goes first to allow runtime model switching
    middleware.append(ConfigurableModelMiddleware())

    # Memory middleware loads AGENTS.md context
    if enable_memory:
        middleware.append(
            MemoryMiddleware(
                sources=[
                    "~/.deepagents/AGENTS.md",
                    "./.deepagents/AGENTS.md",
                ],
                backend=filesystem_backend,
            )
        )

    # Skills middleware loads skill documentation
    if enable_skills:
        middleware.append(
            SkillsMiddleware(
                sources=[
                    "/skills/base/",
                    "/skills/user/",
                ],
                backend=filesystem_backend,
            )
        )

    return middleware
```

### Step 2: Use in Agent Creation

Pass the stack to `create_deep_agent`:

```python
"""agent.py"""

from deepagents import create_deep_agent
from deepagents_cli.configurable_model import ConfigurableModelMiddleware
from deepagents.middleware import MemoryMiddleware, SkillsMiddleware


def create_cli_agent(
    backend,
    enable_memory: bool = True,
    enable_skills: bool = True,
):
    """Create the CLI agent with middleware pipeline."""

    # Build middleware stack
    middleware = build_middleware_stack(
        enable_memory=enable_memory,
        enable_skills=enable_skills,
    )

    # Create agent with middleware
    agent = create_deep_agent(
        backend=backend,
        middleware=middleware,
    )

    return agent
```

## Middleware That Modifies System Prompts

A common pattern is middleware that **injects content into system prompts**. Let's look at how `MemoryMiddleware` does this:

```python
"""Simplified MemoryMiddleware for illustration"""

from langchain.agents.middleware.types import AgentMiddleware
from langchain_core.messages import SystemMessage

from deepagents.middleware._utils import append_to_system_message


class MemoryMiddleware(AgentMiddleware):
    """Load AGENTS.md files and inject context into system prompt."""

    def __init__(self, sources: list[str], backend):
        self.sources = sources
        self.backend = backend

    def wrap_model_call(self, request, handler):
        # Load memory content from sources
        memory_content = self._load_memory()

        if not memory_content:
            return handler(request)  # No memory, pass through

        # Create modified request with injected context
        modified_request = request.copy()

        # Inject into system_prompt string
        modified_request["system_prompt"] = (
            f"<agent_memory>\n{memory_content}\n</agent_memory>\n\n"
            f"{request.get('system_prompt') or ''}"
        )

        # Also update system_message if present
        if request.get("system_message"):
            modified_request["system_message"] = append_to_system_message(
                request["system_message"],
                f"<agent_memory>\n{memory_content}\n</agent_memory>"
            )

        return handler(modified_request)

    def _load_memory(self) -> str:
        """Load and combine memory files."""
        contents = []
        for source in self.sources:
            content = self.backend.read(source)
            if content:
                contents.append(content)
        return "\n\n".join(contents)
```

## The append_to_system_message Utility

The SDK provides a helper for adding content to system messages:

```python
"""From deepagents.middleware._utils"""

from langchain_core.messages import ContentBlock, SystemMessage


def append_to_system_message(
    system_message: SystemMessage | None,
    text: str,
) -> SystemMessage:
    """Append text to a system message.

    Args:
        system_message: Existing system message or None
        text: Text to add

    Returns:
        New SystemMessage with text appended
    """
    new_content: list[ContentBlock] = (
        list(system_message.content_blocks) if system_message else []
    )

    if new_content:
        text = f"\n\n{text}"  # Add separator if existing content

    new_content.append({"type": "text", "text": text})
    return SystemMessage(content_blocks=new_content)
```

## Middleware State

Middleware can maintain state across calls using **state attributes**:

```python
class CachingMiddleware(AgentMiddleware):
    """Example middleware that caches responses."""

    def __init__(self):
        self.cache: dict[str, ModelResponse] = {}

    def wrap_model_call(self, request, handler):
        # Create cache key from request
        cache_key = self._get_cache_key(request)

        # Check cache
        if cache_key in self.cache:
            return self.cache[cache_key]

        # Call handler and cache result
        response = handler(request)
        self.cache[cache_key] = response
        return response
```

For LangGraph integration, middleware can use **private state** that's not persisted:

```python
from typing import Annotated
from langchain.agents.middleware.types import AgentState, PrivateStateAttr


class MyMiddleware(AgentMiddleware):
    """Middleware with private state."""

    # Define state schema
    class State(AgentState):
        my_private_cache: Annotated[dict, PrivateStateAttr]

    def wrap_model_call(self, request, handler):
        # Access private state via request.runtime
        state = request.runtime.state if request.runtime else None
        # ... use state
        return handler(request)
```

## Testing the Pipeline

Test that middleware compose correctly:

```python
# test_pipeline.py
import pytest
from unittest.mock import MagicMock, AsyncMock

from langchain.agents.middleware.types import ModelRequest, ModelResponse
from langchain_core.messages import HumanMessage

from deepagents_cli.configurable_model import ConfigurableModelMiddleware


def test_middleware_chain_order():
    """Test that middleware are called in correct order."""
    call_order = []

    class TracerMiddleware(AgentMiddleware):
        def __init__(self, name):
            self.name = name

        def wrap_model_call(self, request, handler):
            call_order.append(f"before-{self.name}")
            response = handler(request)
            call_order.append(f"after-{self.name}")
            return response

    # Create middleware in order
    middleware = [
        TracerMiddleware("first"),
        TracerMiddleware("second"),
        TracerMiddleware("third"),
    ]

    # Build chain and call
    def build_chain(mw_list, index=0):
        if index == len(mw_list):
            # Terminal handler
            return lambda req: ModelResponse(messages=[])
        mw = mw_list[index]
        def handler(req):
            return build_chain(mw_list, index + 1)(req)
        return lambda req: mw.wrap_model_call(req, handler)

    chain = build_chain(middleware)
    request = ModelRequest(
        model=MagicMock(),
        messages=[HumanMessage(content="test")],
        system_prompt=None,
        system_message=None,
        model_settings={},
        runtime=None,
    )

    chain(request)

    # Verify order: first's before, first's after, etc.
    assert call_order == [
        "before-first",
        "before-second",
        "before-third",
        "after-third",
        "after-second",
        "after-first",
    ]
```

## Common Patterns

### Pattern 1: Conditional Middleware

```python
def build_middleware_stack(config: AppConfig) -> list[AgentMiddleware]:
    middleware = [ConfigurableModelMiddleware()]

    if config.enable_experimental:
        middleware.append(ExperimentalFeatureMiddleware())

    if config.log_level == "debug":
        middleware.append(DebugLoggingMiddleware())

    return middleware
```

### Pattern 2: Middleware with Dependencies

```python
def create_agent_with_middleware(backend):
    # Middleware might need backend reference
    memory_mw = MemoryMiddleware(sources=[...], backend=backend)
    skills_mw = SkillsMiddleware(sources=[...], backend=backend)

    return create_deep_agent(
        middleware=[memory_mw, skills_mw]
    )
```

### Pattern 3: Middleware Factory

```python
def create_model_middleware(config: ModelConfig) -> ConfigurableModelMiddleware:
    """Factory for configurable model middleware with presets."""
    return ConfigurableModelMiddleware(
        default_model=config.default_model,
        max_tokens=config.max_tokens,
    )
```

## Key Takeaways

- **Compose middleware** by passing a list to `create_deep_agent()`
- **Order matters** — Last in list wraps the LLM most closely
- **Each middleware receives** a handler that calls the rest of the chain
- **Request.override()** creates modified copies
- **System prompt injection** is a common pattern
- **State can be maintained** in middleware instances

## Putting It All Together

```python
"""Complete agent setup with middleware pipeline"""

from deepagents import create_deep_agent
from deepagents.middleware import MemoryMiddleware, SkillsMiddleware
from deepagents_cli.configurable_model import ConfigurableModelMiddleware
from deepagents_cli.local_context import LocalContextMiddleware


def create_agent(backend, config: AppConfig):
    """Create agent with full middleware pipeline."""

    middleware = [
        # First: Allow runtime model switching
        ConfigurableModelMiddleware(),

        # Second: Inject local context (git state, project structure)
        LocalContextMiddleware(backend=backend),

        # Third: Load AGENTS.md memory
        MemoryMiddleware(
            sources=[
                "~/.deepagents/AGENTS.md",
                "./.deepagents/AGENTS.md",
            ],
            backend=backend,
        ),

        # Fourth: Load skill documentation
        SkillsMiddleware(
            sources=[
                "/skills/base/",
                "/skills/user/",
            ],
            backend=backend,
        ),
    ]

    return create_deep_agent(
        backend=backend,
        middleware=middleware,
    )
```

## Next Section

[Quiz](./quiz.md) — Test your understanding of middleware concepts.
