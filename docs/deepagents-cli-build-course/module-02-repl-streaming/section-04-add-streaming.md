# Section 4: Add Streaming

Stream tokens in real-time for better UX.

## Update process_message for Streaming

Change from blocking `invoke` to streaming `stream`:

```python
async def process_message_streaming(user_input: str) -> str:
    """Process a message with streaming.
    
    Args:
        user_input: User's message.
    
    Returns:
        Full AI response (for logging/history).
    """
    model = create_model()
    
    messages = [
        SystemMessage(content="You are a helpful coding assistant."),
        HumanMessage(content=user_input),
    ]
    
    # Collect full response
    full_response = ""
    
    # Stream tokens
    for token in model.stream(messages):
        if hasattr(token, "content") and token.content:
            print(token.content, end="", flush=True)
            full_response += token.content
    
    print()  # Newline after response
    return full_response
```

## Why the print inside the loop?

In streaming mode, we want to see tokens as they arrive:

```
>>> What's Python?
What's Python?

Python is a high-level, interpreted programming language known for its simplicity and readability. It was created by Guido van Rossum and first released in 1991.

Key features of Python include:
• Easy-to-read syntax
• Dynamic typing
• Automatic memory management
• Extensive standard library
• Cross-platform compatibility
```

Each line appears as it's generated, not all at once.

## Async Streaming

For truly async streaming with Textual later:

```python
async def process_message_async_streaming(
    user_input: str,
    callback: Callable[[str], None] = None,
) -> str:
    """Process message with async streaming.
    
    Args:
        user_input: User's message.
        callback: Optional callback for each token.
    
    Returns:
        Full response text.
    """
    model = create_model()
    
    messages = [
        SystemMessage(content="You are a helpful coding assistant."),
        HumanMessage(content=user_input),
    ]
    
    full_response = ""
    
    # Async stream - use astream instead of stream
    async for token in model.astream(messages):
        if hasattr(token, "content") and token.content:
            content = token.content
            full_response += content
            if callback:
                callback(content)
    
    return full_response
```

## Complete Streaming REPL

Create `src/deepagents_cli/repl.py`:

```python
"""Async REPL with streaming LLM responses."""

from __future__ import annotations

import asyncio
import sys
from typing import Callable, Optional

from dotenv import load_dotenv

# Load environment variables BEFORE importing anything else
load_dotenv()

from langchain_core.messages import HumanMessage, SystemMessage

from deepagents_cli.models import create_model


async def process_streaming(
    user_input: str,
    *,
    show_progress: bool = True,
) -> str:
    """Process user input with streaming.
    
    Works with any provider: OpenAI, Fireworks, Together AI, etc.
    
    Args:
        user_input: User's message.
        show_progress: Print tokens as they arrive.
    
    Returns:
        Full response text.
    """
    model = create_model()
    
    messages = [
        SystemMessage(content="You are a helpful coding assistant."),
        HumanMessage(content=user_input),
    ]
    
    full_response = ""
    
    # Stream tokens
    for event in model.stream(messages):
        # Each event is a message chunk
        if hasattr(event, "content") and event.content:
            if show_progress:
                print(event.content, end="", flush=True)
            full_response += event.content
    
    if show_progress:
        print()  # Newline after response
    
    return full_response


async def repl(
    system_prompt: str = "You are a helpful coding assistant.",
) -> int:
    """Run the interactive REPL with streaming.
    
    Args:
        system_prompt: System message for the assistant.
    
    Returns:
        Exit code.
    """
    print("Deep Agents CLI - streaming mode")
    print("Type 'exit' or 'quit' to end the session")
    print("-" * 50)
    
    while True:
        try:
            user_input = await asyncio.to_thread(input, ">>> ")
        except (EOFError, KeyboardInterrupt):
            print("\nGoodbye!")
            break
        
        # Handle exit
        if user_input.strip().lower() in ("exit", "quit", "q"):
            print("Goodbye!")
            break
        
        # Skip empty input
        if not user_input.strip():
            continue
        
        # Process with streaming
        try:
            await process_streaming(user_input)
        except Exception as e:
            print(f"\nError: {e}", file=sys.stderr)
    
    return 0


def main() -> int:
    """Entry point."""
    return asyncio.run(repl())
```

## Test Streaming

```bash
uv run python -c "
from deepagents_cli.repl import main
main()
"
```

Try typing questions and watch tokens appear one by one.

## Compare: Invoke vs Stream

```python
# Blocking invoke - waits for full response
response = model.invoke("Write a Python function")
print(response.content)  # All at once, after several seconds

# Streaming - shows tokens as they arrive
for token in model.stream("Write a Python function"):
    print(token.content, end="", flush=True)  # Token by token
```

## Performance Consideration

Streaming has **better perceived latency** but same actual latency:

| Aspect | Invoke | Stream |
|--------|--------|--------|
| Time to first token | ~2-5 seconds | ~0.1 seconds |
| Time to last token | ~2-5 seconds | ~2-5 seconds |
| **Perceived speed** | Slow | Fast |
| **Actual speed** | Same | Same |

## Testing Different Providers

The streaming code works identically across all providers:

### Test with OpenAI
```bash
OPENAI_API_KEY=sk-... uv run python -c "from deepagents_cli.repl import main; main()"
```

### Test with Fireworks AI
```bash
OPENAI_API_KEY=fw-... OPENAI_BASE_URL=https://api.fireworks.ai/inference/v1 uv run python -c "from deepagents_cli.repl import main; main()"
```

### Test with Local Endpoint
```bash
OPENAI_API_KEY=not-needed OPENAI_BASE_URL=http://localhost:8080/v1 uv run python -c "from deepagents_cli.repl import main; main()"
```

## Key Takeaways

- **model.stream()** returns an iterable of message chunks
- **streaming** shows tokens as they're generated
- Each chunk has `.content` with the token text
- **flush=True** ensures immediate display
- Better UX without sacrificing actual speed
- **Works with any provider** — OpenAI, Fireworks, LocalAI, etc.

## Challenge

Modify the REPL to:

1. Accept `--model` argument to specify which model to use
2. Show token count after each response

Hint: Track `len(full_response)` for token count.

## Troubleshooting Streaming

### "Streaming seems slow"

Some providers don't support streaming as well as others:
- **OpenAI**: Excellent streaming support
- **Fireworks**: Good streaming support
- **Local models**: Depends on the model and hardware

### "Tokens appear in chunks, not smoothly"

This is normal! Some providers buffer tokens before sending. OpenAI typically streams smoothly.

### "No output appears until the end"

Check if your provider supports streaming:
```python
# Test if streaming works
for chunk in model.stream("Hi"):
    print(chunk.content, end="")
```

If nothing prints until the end, your provider may not support streaming.

## Next Section

[Quiz](./quiz.md) — Test your understanding of Module 2.
