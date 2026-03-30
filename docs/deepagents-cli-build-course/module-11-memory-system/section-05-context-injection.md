# Section 5: Context Injection

Deep dive into modifying system prompts to inject memory context.

## System Prompt Anatomy

A typical system prompt structure:

```
## System
You are a helpful AI coding assistant.

## Instructions
- Read files when asked
- Execute commands with user approval
- Follow project conventions
```

After memory injection:

```
## System
You are a helpful AI coding assistant.

## Instructions
- Read files when asked
- Execute commands with user approval
- Follow project conventions

## Project Memory
### Environment
- Python 3.11+
- Node.js 20+

### Commands
- `make test` — Run tests
- `make lint` — Run linters
```

## Injection Strategy

Three approaches to consider:

| Approach | Pros | Cons |
|----------|------|------|
| **Append** | Simple, always visible | Can grow too long |
| **Prepend** | Most visible | May be truncated |
| **Dedicated section** | Structured, clear | Requires prompt template |

## Appending to SystemMessage

```python
def inject_memory_into_messages(
    messages: list[BaseMessage],
    memory_content: str,
) -> list[BaseMessage]:
    """Inject memory content into the first SystemMessage found."""
    result = []
    memory_injected = False

    for msg in messages:
        if isinstance(msg, SystemMessage) and not memory_injected:
            enhanced = SystemMessage(
                content=f"{msg.content}\n\n## Project Memory\n{memory_content}"
            )
            result.append(enhanced)
            memory_injected = True
        else:
            result.append(msg)

    if not memory_injected:
        result.insert(0, SystemMessage(content=f"## Project Memory\n{memory_content}"))

    return result
```

## Request Structure

LangChain request structure:

```python
request = {
    "messages": [
        SystemMessage(content="You are helpful."),
        HumanMessage(content="Hello"),
    ],
    "model": "gpt-4o",
    "temperature": 1.0,
}
```

## Handling Multiple SystemMessages

Some prompts have multiple system messages:

```python
def inject_memory_smart(
    messages: list[BaseMessage],
    memory_content: str,
) -> list[BaseMessage]:
    """Inject memory into the primary system message."""
    result = []
    injected = False

    for i, msg in enumerate(messages):
        if isinstance(msg, SystemMessage):
            if not injected:
                if i == 0 or "You are" in msg.content:
                    enhanced = SystemMessage(
                        content=f"{msg.content}\n\n## Project Memory\n{memory_content}"
                    )
                    result.append(enhanced)
                    injected = True
                else:
                    result.append(msg)
            else:
                result.append(msg)
        else:
            result.append(msg)

    if not injected:
        result.insert(0, SystemMessage(content=f"## Project Memory\n{memory_content}"))

    return result
```

## Token Budget Management

Memory can consume significant tokens. Implement limits:

```python
MAX_MEMORY_TOKENS = 2000

def truncate_memory(content: str, max_tokens: int = MAX_MEMORY_TOKENS) -> str:
    """Truncate memory to fit token budget."""
    approx_chars = max_tokens * 4
    
    if len(content) <= approx_chars:
        return content
    
    truncated = content[:approx_chars]
    
    last_newline = truncated.rfind("\n")
    if last_newline > approx_chars * 0.8:
        truncated = truncated[:last_newline]
    
    return truncated + "\n\n[Memory truncated due to length]"
```

## Custom Injection Points

For more control, use a template marker:

```python
DEFAULT_SYSTEM_TEMPLATE = """You are a helpful AI coding assistant.

## Instructions
- Be concise and accurate
- Ask clarifying questions when needed

[MEMORY_PLACEHOLDER]

## User Request
{user_message}"""

def inject_into_template(
    template: str,
    memory_content: str,
) -> str:
    """Inject memory into a template with placeholder."""
    placeholder = "[MEMORY_PLACEHOLDER]"
    
    if placeholder not in template:
        return f"{template}\n\n## Project Memory\n{memory_content}"
    
    injection = f"## Project Memory\n{memory_content}"
    return template.replace(placeholder, injection)
```

## Middleware Integration

Complete MemoryMiddleware implementation:

```python
class MemoryMiddleware(AgentMiddleware):
    def __init__(
        self,
        get_memory: Callable[[], Optional[str]],
        priority: int = 100,
        max_tokens: int = 2000,
    ):
        super().__init__(priority=priority)
        self.get_memory = get_memory
        self.max_tokens = max_tokens

    async def wrap_model_call(self, request, handler):
        memory_content = self.get_memory()
        
        if not memory_content:
            return await handler(request)

        memory_content = truncate_memory(memory_content, self.max_tokens)

        messages = request["messages"]
        enhanced_messages = inject_memory_into_messages(messages, memory_content)
        request["messages"] = enhanced_messages

        return await handler(request)
```

## Debugging Injection

Add logging to verify injection:

```python
async def wrap_model_call(self, request, handler):
    memory_content = self.get_memory()
    
    if memory_content:
        original_len = len(request["messages"][0].content)
        enhanced_messages = inject_memory_into_messages(
            request["messages"], 
            memory_content
        )
        new_len = len(enhanced_messages[0].content)
        
        print(f"[memory] Injected {new_len - original_len} chars "
              f"({original_len} → {new_len})")
        
        request["messages"] = enhanced_messages

    return await handler(request)
```

## Testing Injection

```python
@pytest.mark.asyncio
async def test_injects_into_system_message():
    middleware = MemoryMiddleware(
        get_memory=lambda: "Use make test",
        priority=100,
    )

    request = {
        "messages": [
            SystemMessage(content="You are helpful."),
            HumanMessage(content="Run tests"),
        ],
    }

    result = await middleware.wrap_model_call(request, lambda r: r)

    assert "## Project Memory" in result["messages"][0].content
    assert "make test" in result["messages"][0].content
```

## Key Takeaways

- **Injection point** — First SystemMessage is the target
- **Multiple system messages** — Find the primary one
- **Token budget** — Truncate to prevent context overflow
- **Template markers** — Enable precise injection points
- **Logging** — Verify injection for debugging

## Next Section

[Quiz](./quiz.md) — Test your understanding of the memory system.
