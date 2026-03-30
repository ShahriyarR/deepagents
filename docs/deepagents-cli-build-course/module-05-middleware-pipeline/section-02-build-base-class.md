# Section 2: Build the Base Class

Create a minimal AgentMiddleware subclass that compiles.

## The AgentMiddleware Base

LangChain Agents provides `AgentMiddleware` as a base class:

```python
from langchain.agents.middleware.types import AgentMiddleware

class MyMiddleware(AgentMiddleware):
    pass
```

That's it! A minimal middleware that does nothing. But to be useful, you need to implement the hooks.

## The Hook Methods

`AgentMiddleware` provides two hooks:

```python
class AgentMiddleware:
    def wrap_model_call(
        self,
        request: ModelRequest[ContextT],
        handler: Callable[[ModelRequest[ContextT]], ModelResponse[ResponseT]],
    ) -> ModelResponse[ResponseT]:
        """Synchronous hook for intercepting LLM calls."""
        return handler(request)

    async def awrap_model_call(
        self,
        request: ModelRequest[ContextT],
        handler: Callable[[ModelRequest[ContextT]], Awaitable[ModelResponse[ResponseT]]],
    ) -> ModelResponse[ResponseT]:
        """Async hook for intercepting LLM calls."""
        return await handler(request)
```

Both hooks receive:
- `request` — The ModelRequest being sent to the LLM
- `handler` — The next handler in the chain (or the LLM itself)

## The ModelRequest Structure

Let's look at what a `ModelRequest` contains:

```python
from langchain.agents.middleware.types import ModelRequest

# ModelRequest is a TypedDict with:
class ModelRequest(TypedDict):
    model: BaseChatModel              # The LLM being called
    messages: list[BaseMessage]       # Conversation messages
    system_message: SystemMessage | None  # System prompt
    system_prompt: str | None         # Raw system prompt string
    model_settings: dict[str, Any]    # Model-specific settings
    runtime: Runtime | None           # LangGraph runtime context
```

## Your First Middleware

Let's build a `LoggingMiddleware` that logs requests:

```python
import logging
from typing import Any

from langchain.agents.middleware.types import (
    AgentMiddleware,
    ModelRequest,
    ModelResponse,
)

logger = logging.getLogger(__name__)


class LoggingMiddleware(AgentMiddleware):
    """Log all LLM requests and responses."""

    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Any,  # Callable signature varies
    ) -> ModelResponse:
        # Log the request
        logger.info(f"LLM Request: {request['system_prompt'][:100]}...")
        
        # Call the next handler
        response = handler(request)
        
        # Log the response
        logger.info(f"LLM Response: {response}")
        
        return response
```

## Type Variables Explained

You might see `ModelRequest[ContextT]` — these are type variables:

```python
from typing import TypeVar

ContextT = TypeVar("ContextT")  # Runtime context type
ResponseT = TypeVar("ResponseT")  # Response type

class ModelRequest(TypedDict, Generic[ContextT]):
    # ...
```

For most cases, you can use `ModelRequest` without parameters:

```python
def wrap_model_call(
    self,
    request: ModelRequest,  # Works fine
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    ...
```

## Complete Minimal Middleware

Here's a complete, working middleware that modifies the system prompt:

```python
"""middleware/example_middleware.py"""

from typing import Any, Callable

from langchain.agents.middleware.types import (
    AgentMiddleware,
    ModelRequest,
    ModelResponse,
)
from langchain_core.messages import SystemMessage


class PrefixMiddleware(AgentMiddleware):
    """Add a prefix to every system prompt."""

    def __init__(self, prefix: str):
        self.prefix = prefix

    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        # Create a modified request with prefixed system prompt
        current_prompt = request.get("system_prompt") or ""
        modified_request = request.copy()
        modified_request["system_prompt"] = f"{self.prefix}\n\n{current_prompt}"
        
        # Also update system_message if present
        if request.get("system_message"):
            current_message = request["system_message"]
            new_content = f"{self.prefix}\n\n{current_message.content_blocks[0]['text']}"
            modified_request["system_message"] = SystemMessage(content_blocks=[
                {"type": "text", "text": new_content}
            ])
        
        # Pass to next handler
        return handler(modified_request)
```

## Testing Your Middleware

Let's write a quick test to verify it works:

```python
# test_middleware.py
import pytest
from unittest.mock import MagicMock

from langchain.agents.middleware.types import ModelRequest, ModelResponse
from langchain_core.messages import HumanMessage, SystemMessage

# Import your middleware
from my_middleware import PrefixMiddleware


def test_prefix_middleware_adds_prefix():
    """Test that PrefixMiddleware adds prefix to system prompt."""
    # Create a mock handler that returns a mock response
    handler = MagicMock(return_value=ModelResponse(messages=[]))
    
    # Create middleware with a prefix
    middleware = PrefixMiddleware(prefix="[PREFIX]")
    
    # Create a sample request
    request = ModelRequest(
        model=MagicMock(),
        messages=[HumanMessage(content="Hello")],
        system_prompt="You are a helpful assistant.",
        system_message=SystemMessage(content_blocks=[
            {"type": "text", "text": "You are a helpful assistant."}
        ]),
        model_settings={},
        runtime=None,
    )
    
    # Call the middleware
    response = middleware.wrap_model_call(request, handler)
    
    # Verify handler was called
    handler.assert_called_once()
    
    # Get the modified request that was passed to handler
    modified_request = handler.call_args[0][0]
    
    # Verify prefix was added
    assert "[PREFIX]" in modified_request["system_prompt"]
    assert modified_request["system_prompt"].startswith("[PREFIX]")


def test_prefix_middleware_passes_through_when_no_system_prompt():
    """Test that middleware works when there's no system prompt."""
    handler = MagicMock(return_value=ModelResponse(messages=[]))
    middleware = PrefixMiddleware(prefix="[PREFIX]")
    
    request = ModelRequest(
        model=MagicMock(),
        messages=[HumanMessage(content="Hello")],
        system_prompt=None,
        system_message=None,
        model_settings={},
        runtime=None,
    )
    
    response = middleware.wrap_model_call(request, handler)
    
    # Handler should still be called
    handler.assert_called_once()
```

## Key Takeaways

- **AgentMiddleware** is the base class for all middleware
- **`wrap_model_call()`** is the sync hook for intercepting requests
- **`awrap_model_call()`** is the async version
- **Handler pattern** — Call `handler(request)` to continue the chain
- **Request is a TypedDict** — Can be copied and modified with `.copy()` and `override()`

## Next Section

[Implement wrap_model_call()](./section-03-implement-wrap.md) — Build the ConfigurableModelMiddleware with request modification.
