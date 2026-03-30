# Section 3: Implement wrap_model_call()

Build ConfigurableModelMiddleware for runtime model switching.

## The Goal

We want a middleware that can **swap the model at runtime** without recompiling the graph. This is useful for:

- Switching between fast/cheap and slow/powerful models
- A/B testing different models
- User preference overrides

## The Approach

We'll modify the `request` before passing it to the handler:

```python
def wrap_model_call(self, request, handler):
    # 1. Check if we should override the model
    # 2. Create a new request with the overridden model
    # 3. Pass to handler
    return handler(modified_request)
```

## Step 1: Basic Structure

Start with the class skeleton:

```python
"""configurable_model.py"""

from typing import Any

from langchain.agents.middleware.types import (
    AgentMiddleware,
    ModelRequest,
    ModelResponse,
)

import logging

logger = logging.getLogger(__name__)


class ConfigurableModelMiddleware(AgentMiddleware):
    """Swap the model at runtime from runtime context."""

    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Any,  # Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        return handler(request)  # Pass through unchanged for now
```

## Step 2: Access Runtime Context

The model override comes from `runtime.context`:

```python
def _apply_overrides(self, request: ModelRequest) -> ModelRequest:
    """Apply model overrides from runtime context."""
    runtime = request.runtime
    
    # No runtime means no overrides
    if runtime is None:
        return request
    
    # Context is where callers pass data
    ctx = runtime.context
    if not isinstance(ctx, dict):
        return request
    
    # Check for model override
    model_override = ctx.get("model")
    if model_override:
        logger.debug("Overriding model to %s", model_override)
        # Create new request with overridden model
        return request.override(model=model_override)
    
    return request
```

## Step 3: Override Request

The `request.override()` method creates a modified copy:

```python
def wrap_model_call(
    self,
    request: ModelRequest,
    handler: Any,
) -> ModelResponse:
    # Apply any overrides
    modified_request = self._apply_overrides(request)
    
    # Call next handler with (possibly) modified request
    return handler(modified_request)
```

## Complete Implementation

Here's the full `ConfigurableModelMiddleware`:

```python
"""configurable_model.py"""

from __future__ import annotations

import logging
from typing import TYPE_CHECKING, Any

if TYPE_CHECKING:
    from collections.abc import Awaitable, Callable


logger = logging.getLogger(__name__)


class ConfigurableModelMiddleware(AgentMiddleware):
    """Swap the model or per-call settings from `runtime.context`.

    This middleware reads from `runtime.context` which can be passed via:
        agent.astream({"messages": [...]}, context={"model": "gpt-4o"})

    Usage:
        # In your agent setup
        agent = create_deep_agent(middleware=[ConfigurableModelMiddleware()])

        # At runtime, switch models
        result = await agent.ainvoke(
            {"messages": [...]},
            context={"model": "gpt-4o"}
        )
    """

    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        """Apply runtime overrides and delegate to the next handler."""
        modified_request = self._apply_overrides(request)
        return handler(modified_request)

    async def awrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], Awaitable[ModelResponse]],
    ) -> ModelResponse:
        """Apply runtime overrides and delegate to the next async handler."""
        modified_request = self._apply_overrides(request)
        return await handler(modified_request)

    def _apply_overrides(self, request: ModelRequest) -> ModelRequest:
        """Apply model/param overrides from `runtime.context`.

        Args:
            request: Original ModelRequest

        Returns:
            Modified request with overrides applied, or original if no overrides
        """
        runtime = request.runtime
        if runtime is None:
            return request

        ctx = runtime.context
        if not isinstance(ctx, dict):
            return request

        overrides: dict[str, Any] = {}

        # Model swap
        new_model = ctx.get("model")
        if new_model:
            overrides["model"] = new_model

        # Model param merge (temperature, max_tokens, etc.)
        model_params = ctx.get("model_params", {})
        if model_params:
            overrides["model_settings"] = {**request.model_settings, **model_params}

        if not overrides:
            return request

        return request.override(**overrides)
```

## How It Works

1. **Runtime context** — Passed when calling `agent.ainvoke()` or `agent.astream()`
2. **Override check** — Looks for `model` or `model_params` in context
3. **Request.override()** — Creates a modified copy of the request
4. **Handler call** — Passes the modified request to the next middleware/LLM

## Usage Example

```python
# Setup the agent with middleware
from deepagents import create_deep_agent

agent = create_deep_agent(middleware=[ConfigurableModelMiddleware()])

# Use default model (configured at startup)
result = await agent.ainvoke({"messages": [...]})

# Override to a different model for this specific call
result = await agent.ainvoke(
    {"messages": [...]},
    context={"model": "gpt-4o"}
)

# Override model params too
result = await agent.ainvoke(
    {"messages": [...]},
    context={
        "model": "gpt-4o",
        "model_params": {"temperature": 0.0}  # More deterministic
    }
)
```

## Adding Model Resolution

For a real CLI, you'd want to resolve model names to actual chat models:

```python
from deepagents_cli.config import create_model
from deepagents_cli.model_config import ModelConfigError


def _apply_overrides(self, request: ModelRequest) -> ModelRequest:
    runtime = request.runtime
    if runtime is None:
        return request

    ctx = runtime.context
    if not isinstance(ctx, dict):
        return request

    overrides: dict[str, Any] = {}

    # Resolve model name to actual model instance
    new_model_name = ctx.get("model")
    if new_model_name:
        try:
            model_result = create_model(new_model_name)
            overrides["model"] = model_result.model
            logger.debug("Resolved model override to %s", model_result.model_name)
        except ModelConfigError:
            logger.exception("Failed to resolve model '%s'", new_model_name)
            return request  # Skip override on error

    # Merge model settings
    model_params = ctx.get("model_params", {})
    if model_params:
        overrides["model_settings"] = {**request.model_settings, **model_params}

    if not overrides:
        return request

    return request.override(**overrides)
```

## Testing the Middleware

```python
# test_configurable_model.py
import pytest
from unittest.mock import MagicMock

from langchain.agents.middleware.types import ModelRequest, ModelResponse
from langchain_core.messages import HumanMessage

from configurable_model import ConfigurableModelMiddleware


def test_passes_through_without_context():
    """Test that middleware doesn't modify request when no context."""
    handler = MagicMock(return_value=ModelResponse(messages=[]))
    middleware = ConfigurableModelMiddleware()

    request = ModelRequest(
        model=MagicMock(),
        messages=[HumanMessage(content="Hello")],
        system_prompt=None,
        system_message=None,
        model_settings={},
        runtime=None,
    )

    middleware.wrap_model_call(request, handler)
    
    # Original request passed through
    assert handler.call_args[0][0] is request


def test_applies_model_override():
    """Test that middleware applies model override from context."""
    handler = MagicMock(return_value=ModelResponse(messages=[]))
    middleware = ConfigurableModelMiddleware()

    # Create runtime with context
    runtime = MagicMock()
    runtime.context = {"model": "gpt-4o"}

    request = ModelRequest(
        model=MagicMock(),
        messages=[HumanMessage(content="Hello")],
        system_prompt=None,
        system_message=None,
        model_settings={},
        runtime=runtime,
    )

    middleware.wrap_model_call(request, handler)
    
    # Handler should be called with modified request
    modified = handler.call_args[0][0]
    assert modified is not request
    assert modified["model"] == "gpt-4o"


def test_merges_model_settings():
    """Test that middleware merges model_settings."""
    handler = MagicMock(return_value=ModelResponse(messages=[]))
    middleware = ConfigurableModelMiddleware()

    runtime = MagicMock()
    runtime.context = {"model_params": {"temperature": 0.0}}

    request = ModelRequest(
        model=MagicMock(),
        messages=[HumanMessage(content="Hello")],
        system_prompt=None,
        system_message=None,
        model_settings={"max_tokens": 1000},
        runtime=runtime,
    )

    middleware.wrap_model_call(request, handler)
    
    modified = handler.call_args[0][0]
    assert modified["model_settings"]["temperature"] == 0.0
    assert modified["model_settings"]["max_tokens"] == 1000
```

## Key Takeaways

- **`wrap_model_call()`** receives request and handler
- **`request.override(**changes)`** creates a modified copy
- **`runtime.context`** carries per-call data
- **Always call handler** with the (possibly modified) request
- **Both sync and async** — Implement both hooks for completeness

## Next Section

[Compose Middleware](./section-04-compose-middleware.md) — Stack multiple middleware and build the pipeline.
