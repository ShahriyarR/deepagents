# Section 3: MemoryMiddleware

Build middleware to load AGENTS.md and inject context into LLM requests.

## Middleware Architecture

Recall from Module 5: Middleware wraps LLM calls to intercept and modify requests:

```
Request → MemoryMiddleware → SkillsMiddleware → LLM
        ← MemoryMiddleware ← SkillsMiddleware ← LLM
```

MemoryMiddleware sits early in the chain to add project context before any other processing.

## Creating MemoryMiddleware

```python
from langchain.agents.middleware_types import AgentMiddleware
from langchain.schema.messages import SystemMessage
from pathlib import Path
from typing import Optional

class MemoryMiddleware(AgentMiddleware):
    """Middleware that injects AGENTS.md context into system prompt."""

    def __init__(
        self,
        memory_content: Optional[str] = None,
        priority: int = 100,  # Lower = runs first
    ):
        super().__init__(priority=priority)
        self.memory_content = memory_content

    async def wrap_model_call(self, request, handler):
        # 1. Load memory if not already loaded
        if self.memory_content is None:
            return await handler(request)

        # 2. Modify request to inject memory
        modified_request = self._inject_memory(request, self.memory_content)

        # 3. Pass to next middleware/LLM
        return await handler(modified_request)

    def _inject_memory(self, request, memory_content: str) -> request:
        # Implementation in next section
        ...
```

## Request Modification

The request object contains messages. Find the SystemMessage and append memory:

```python
def _inject_memory(self, request, memory_content: str):
    messages = list(request["messages"])
    
    for i, msg in enumerate(messages):
        if isinstance(msg, SystemMessage):
            original_content = msg.content
            new_content = f"{original_content}\n\n## Project Memory\n{memory_content}"
            messages[i] = SystemMessage(content=new_content)
            break
    
    request["messages"] = messages
    return request
```

## Lazy Loading Pattern

Load memory only when needed:

```python
class MemoryMiddleware(AgentMiddleware):
    def __init__(
        self,
        memory_loader: Optional[Callable[[], Optional[str]]] = None,
        priority: int = 100,
    ):
        super().__init__(priority=priority)
        self._memory_loader = memory_loader
        self._memory_content: Optional[str] = None

    @property
    def memory_content(self) -> Optional[str]:
        if self._memory_content is None and self._memory_loader:
            self._memory_content = self._memory_loader()
        return self._memory_content

    async def wrap_model_call(self, request, handler):
        if not self.memory_content:
            return await handler(request)
        
        modified_request = self._inject_memory(request, self.memory_content)
        return await handler(modified_request)
```

## Usage with App

Wire up middleware when building the agent:

```python
def create_middleware_stack(
    backend: FilesystemBackend,
    root_path: Path,
) -> list[AgentMiddleware]:
    def load_memory() -> Optional[str]:
        return load_memory_content(root_path, backend)

    return [
        MemoryMiddleware(memory_loader=load_memory, priority=100),
        LoggingMiddleware(priority=90),
        SkillsMiddleware(priority=80),
    ]
```

## Middleware Ordering

Memory should load early (low priority) so other middleware see the enriched context:

```
Priority 100: MemoryMiddleware      ← Runs first on request, last on response
Priority 90:  LoggingMiddleware
Priority 80:  SkillsMiddleware      ← Sees memory-enhanced system prompt
Priority 0:   LLM                   ← Actual LLM call
```

## Handling Missing Memory

No AGENTS.md? No problem — middleware should be optional:

```python
async def wrap_model_call(self, request, handler):
    # Skip injection if no memory
    if not self.memory_content:
        return await handler(request)
    
    # Memory exists, inject and continue
    modified_request = self._inject_memory(request, self.memory_content)
    return await handler(modified_request)
```

## Testing MemoryMiddleware

Mock the loader to test injection:

```python
import pytest

@pytest.mark.asyncio
async def test_memory_middleware_injects_content():
    memory_loader = lambda: "Use make test for testing"
    
    middleware = MemoryMiddleware(
        memory_loader=memory_loader,
        priority=100,
    )
    
    request = {
        "messages": [
            SystemMessage(content="You are a helpful assistant.")
        ]
    }
    
    modified = await middleware.wrap_model_call(
        request,
        lambda r: r  # Passthrough handler
    )
    
    assert "## Project Memory" in modified["messages"][0].content
    assert "make test" in modified["messages"][0].content
```

## Key Takeaways

- **MemoryMiddleware** wraps LLM calls to inject project context
- **SystemMessage modification** appends memory to existing system prompt
- **Lazy loading** defers file I/O until first use
- **Middleware ordering** matters — Memory runs early
- **Graceful handling** when AGENTS.md doesn't exist

## Next Section

[Load AGENTS.md](./section-04-load-agents-md.md) — Implement the file loading and parsing logic.
