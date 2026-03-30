# Section 3: Connect the LLM

Integrate LangChain with any OpenAI-compatible LLM provider.

## Multi-Provider Support

The CLI supports multiple LLM providers through a unified interface:

- **OpenAI** (default) — GPT-4o, GPT-4, GPT-3.5-turbo
- **OpenAI-Compatible APIs** — OpenCode, Fireworks AI, Together AI, LocalAI, etc.
- **Anthropic** — Claude (optional, requires extra dependency)

## Set Up API Key

Create `src/deepagents_cli/config.py` for configuration:

```python
"""Configuration for Deep Agents CLI."""

from __future__ import annotations

import os
from pathlib import Path
from typing import Optional


class Config:
    """Configuration loaded from environment and files."""
    
    @property
    def llm_config(self) -> dict:
        """Get complete LLM configuration.
        
        Format: provider:model (e.g., openai:gpt-4o)
        """
        model_str = os.environ.get("DEEPAGENTS_MODEL", "openai:gpt-4o")
        
        if ":" not in model_str:
            # Default to openai if no provider specified
            provider = "openai"
            model = model_str
        else:
            provider, model = model_str.split(":", 1)
        
        # Get provider-specific environment variables
        provider_prefix = provider.upper()
        
        return {
            "provider": provider,
            "model": model,
            "api_key": os.environ.get(f"{provider_prefix}_API_KEY"),
            "base_url": os.environ.get(f"{provider_prefix}_BASE_URL"),
        }
    
    @property
    def openai_api_key(self) -> Optional[str]:
        """Get OpenAI API key from environment."""
        return os.environ.get("OPENAI_API_KEY")
    
    @property
    def default_model(self) -> str:
        """Get default model."""
        return os.environ.get("DEEPAGENTS_MODEL", "gpt-4o")


# Global config instance
config = Config()
```

## Create Model Factory

Create `src/deepagents_cli/models.py`:

```python
"""Model factory for creating LLM instances."""

from __future__ import annotations

from typing import Optional, Union

from langchain_openai import ChatOpenAI


def create_model(
    model: Optional[str] = None,
    api_key: Optional[str] = None,
    base_url: Optional[str] = None,
) -> ChatOpenAI:
    """Create a ChatOpenAI model instance.
    
    Works with any OpenAI-compatible API including:
    - OpenAI (default)
    - OpenCode
    - Fireworks AI
    - Together AI
    - LocalAI
    
    Args:
        model: Model name (e.g., "gpt-4o" or "accounts/fireworks/models/llama-v3p1-70b-instruct").
               Defaults to Config default.
        api_key: API key. Defaults to environment variable.
        base_url: Custom API endpoint URL. Defaults to environment variable.
    
    Returns:
        Configured ChatOpenAI instance.
    """
    from deepagents_cli.config import config
    
    cfg = config.llm_config
    
    return ChatOpenAI(
        model=model or cfg["model"],
        api_key=api_key or cfg["api_key"],
        base_url=base_url or cfg.get("base_url"),
        timeout=30,
    )


def create_anthropic_model(
    model: Optional[str] = None,
    api_key: Optional[str] = None,
):
    """Create an Anthropic Claude model (requires langchain-anthropic).
    
    Install with: uv add langchain-anthropic
    
    Args:
        model: Model name (e.g., "claude-3-sonnet-20240229").
        api_key: Anthropic API key.
    """
    try:
        from langchain_anthropic import ChatAnthropic
    except ImportError:
        raise ImportError(
            "langchain-anthropic not installed. "
            "Run: uv add langchain-anthropic"
        )
    
    from deepagents_cli.config import config
    
    return ChatAnthropic(
        model=model or "claude-3-sonnet-20240229",
        api_key=api_key or os.environ.get("ANTHROPIC_API_KEY"),
        timeout=30,
    )
```

## Create .env File

In your project root:

```bash
# .env - NEVER commit this file to git!
OPENAI_API_KEY=sk-your-key-here
```

Add to `.gitignore`:
```
.env
```

## Test Model Connection

Create `src/deepagents_cli/test_model.py`:

```python
"""Test model connection."""

from dotenv import load_dotenv

# Load environment variables from .env
load_dotenv()

from deepagents_cli.models import create_model


def main() -> int:
    """Test the model connection."""
    print("Testing model connection...")
    
    try:
        model = create_model()
        print(f"Model: {model.model_name}")
        print(f"API Key set: {bool(model.openai_api_key)}")
        
        # Test invoke
        print("\nTesting invoke (blocking)...")
        response = model.invoke("Say 'Hello, World!' in exactly those words.")
        print(f"Response: {response.content}")
        
        return 0
        
    except Exception as e:
        print(f"Error: {e}")
        return 1


if __name__ == "__main__":
    raise SystemExit(main())
```

## Run the Test

```bash
uv run python src/deepagents_cli/test_model.py
```

Expected output:
```
Testing model connection...
Model: gpt-4o
API Key set: True

Testing invoke (blocking)...
Response: Hello, World!
```

## Using Custom OpenAI-Compatible APIs

You can use any OpenAI-compatible API by setting the base URL:

### OpenCode

```bash
# .env
OPENAI_API_KEY=your-opencode-key
OPENAI_BASE_URL=https://api.opencode.ai/v1
DEEPAGENTS_MODEL=openai:gpt-4o
```

### Fireworks AI

```bash
# .env
OPENAI_API_KEY=your-fireworks-key
OPENAI_BASE_URL=https://api.fireworks.ai/inference/v1
DEEPAGENTS_MODEL=openai:accounts/fireworks/models/llama-v3p1-70b-instruct
```

### Together AI

```bash
# .env
OPENAI_API_KEY=your-together-key
OPENAI_BASE_URL=https://api.together.xyz/v1
DEEPAGENTS_MODEL=openai:meta-llama/Llama-3-70b-chat-hf
```

### Local Endpoint (e.g., LocalAI, Ollama)

```bash
# .env
OPENAI_API_KEY=not-needed  # or any placeholder
OPENAI_BASE_URL=http://localhost:8080/v1
DEEPAGENTS_MODEL=openai:gpt-4  # model name your local server expects
```

## LangChain Message Types

LangChain uses typed messages:

```python
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage

# User message
HumanMessage(content="Hello!")

# AI response
AIMessage(content="Hi there!")

# System instructions (sets AI behavior)
SystemMessage(content="You are a helpful assistant.")
```

## Build Conversation

```python
from langchain_core.messages import HumanMessage, SystemMessage
from deepagents_cli.models import create_model

# Create model
model = create_model()

# Build messages
messages = [
    SystemMessage(content="You are a helpful coding assistant."),
    HumanMessage(content="What is 2+2?"),
]

# Get response
response = model.invoke(messages)

# response.content contains the text
print(response.content)  # "4"
```

## Update REPL to Use Model

```python
"""Async REPL with LLM integration."""

import asyncio
from typing import Callable, Awaitable

from dotenv import load_dotenv

# Load environment variables before anything else
load_dotenv()

from langchain_core.messages import HumanMessage, SystemMessage

from deepagents_cli.models import create_model


async def process_message(user_input: str) -> str:
    """Process a message with the LLM.
    
    Args:
        user_input: User's message.
    
    Returns:
        AI response.
    """
    model = create_model()
    
    # Simple conversation (no history yet)
    messages = [
        SystemMessage(content="You are a helpful coding assistant."),
        HumanMessage(content=user_input),
    ]
    
    response = model.invoke(messages)
    return response.content


async def repl() -> int:
    """Run the REPL with LLM integration."""
    print("Deep Agents CLI - type 'exit' to quit")
    print("-" * 40)
    
    while True:
        try:
            user_input = await asyncio.to_thread(input, ">>> ")
        except (EOFError, KeyboardInterrupt):
            print("\nGoodbye!")
            break
        
        if user_input.strip().lower() in ("exit", "quit", "q"):
            print("Goodbye!")
            break
        
        if not user_input.strip():
            continue
        
        # Process with LLM
        try:
            response = await process_message(user_input)
            print(response)
        except Exception as e:
            print(f"Error: {e}")
    
    return 0
```

## Test It

```bash
uv run python -c "
import asyncio
from deepagents_cli.repl import repl
asyncio.run(repl())
"
```

## Key Takeaways

- **Multi-provider support** — Works with any OpenAI-compatible API
- **Environment variables** — Store API keys securely in `.env`
- **BASE_URL** — Switch providers by changing the endpoint
- **create_model()** factory — Automatically configures the right provider
- **LangChain messages** — HumanMessage, AIMessage, SystemMessage
- **NEVER hardcode keys** — Always use environment variables

## Troubleshooting

### "API key not found"

Make sure you've:
1. Created `.env` file
2. Added `load_dotenv()` at the start of your program
3. Added `.env` to `.gitignore`

### "Connection refused" or timeout

Check your `OPENAI_BASE_URL`:
```bash
# Test the endpoint
curl $OPENAI_BASE_URL/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

### "Model not found"

Different providers use different model names:
- OpenAI: `gpt-4o`, `gpt-4`, `gpt-3.5-turbo`
- Fireworks: `accounts/fireworks/models/llama-v3p1-70b-instruct`
- Together: `meta-llama/Llama-3-70b-chat-hf`

## Next Section

[Add Streaming](./section-04-add-streaming.md) — Stream tokens in real-time.
