# Section 2: Build the REPL Skeleton

Create the basic input loop structure.

## What is a REPL?

A **REPL** (Read-Eval-Print Loop) is a simple interactive environment:

```
┌─────────────────────────────────────┐
│           READ                       │
│   Get input from user               │
└─────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│           EVAL                       │
│   Process the input                  │
└─────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│           PRINT                      │
│   Show the output                    │
└─────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│           LOOP                       │
│   Back to READ                       │
└─────────────────────────────────────┘
```

## Simple Sync REPL

```python
"""Simple REPL example."""

def repl():
    """Simple read-eval-print loop."""
    print("Simple REPL - type 'exit' to quit")
    
    while True:
        # Read
        user_input = input(">>> ")
        
        # Check for exit
        if user_input.strip().lower() == "exit":
            print("Goodbye!")
            break
        
        # Eval (placeholder)
        print(f"Echo: {user_input}")
        
        # Print (implicit)

if __name__ == "__main__":
    repl()
```

## Test It

```bash
uv run python src/deepagents_cli/repl.py
```

Output:
```
Simple REPL - type 'exit' to quit
>>> Hello
Echo: Hello
>>> exit
Goodbye!
```

## Async REPL

For our CLI, we need **async** because streaming LLM responses is async:

```python
"""Async REPL for streaming responses."""

import asyncio


async def repl():
    """Async read-eval-print loop."""
    print("Async REPL - type 'exit' to quit")
    
    while True:
        # Read input (sync, but we're in async context)
        user_input = await asyncio.to_thread(input, ">>> ")
        
        # Check for exit
        if user_input.strip().lower() == "exit":
            print("Goodbye!")
            break
        
        # For now, just echo
        print(f"Echo: {user_input}")

    return 0


def main() -> int:
    """Entry point."""
    return asyncio.run(repl())


if __name__ == "__main__":
    raise SystemExit(main())
```

## Key Async Patterns

```python
# Run async code
asyncio.run(repl())

# Convert sync code to async
result = await asyncio.to_thread(sync_function, arg)

# Sleep (non-blocking)
await asyncio.sleep(0.1)
```

## Why asyncio.to_thread for input()?

`input()` is **sync** — it blocks the thread. In an async context, we don't want to block the event loop:

```python
# WRONG - blocks the event loop
user_input = input(">>> ")

# RIGHT - runs in thread pool, doesn't block
user_input = await asyncio.to_thread(input, ">>> ")
```

## Create repl.py

Create `src/deepagents_cli/repl.py`:

```python
"""Async REPL for Deep Agents CLI."""

import asyncio
from typing import Callable, Awaitable


async def repl(
    process_fn: Callable[[str], Awaitable[str]] = None,
) -> int:
    """Run the interactive REPL.
    
    Args:
        process_fn: Optional async function to process input.
                     If None, just echoes input.
    
    Returns:
        Exit code.
    """
    print("Deep Agents CLI - type 'exit' to quit")
    print("-" * 40)
    
    while True:
        try:
            # Read input (non-blocking)
            user_input = await asyncio.to_thread(input, ">>> ")
            
        except (EOFError, KeyboardInterrupt):
            print("\nGoodbye!")
            break
        
        # Check for exit
        if user_input.strip().lower() in ("exit", "quit", "q"):
            print("Goodbye!")
            break
        
        # Skip empty input
        if not user_input.strip():
            continue
        
        # Process input
        if process_fn:
            try:
                response = await process_fn(user_input)
                print(response)
            except Exception as e:
                print(f"Error: {e}")
        else:
            print(f"Echo: {user_input}")
    
    return 0


def main() -> int:
    """Entry point."""
    return asyncio.run(repl())
```

## Test the REPL

```bash
uv run python -c "
from deepagents_cli.repl import repl
asyncio.run(repl())
"
```

## Key Takeaways

- **REPL** = Read-Eval-Print Loop
- Use **async/await** for I/O operations that can block
- **asyncio.to_thread()** runs sync functions without blocking
- Handle **EOFError** and **KeyboardInterrupt** for clean exits

## Next Section

[Connect the LLM](./section-03-connect-llm.md) — Add LangChain + Anthropic.
