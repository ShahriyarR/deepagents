# Section 3: Configure Dependencies

Add LangChain, LangGraph, Textual, and OpenAI to your project.

## The Dependencies We Need

| Package | Purpose | Version |
|---------|---------|---------|
| `langchain` | LLM abstractions | >=1.2.13 |
| `langgraph` | Agent graph framework | >=1.1.2 |
| `langchain-openai` | OpenAI GPT models | >=1.3.5 |
| `textual` | TUI framework | >=8.0.0 |
| `openai` | OpenAI API client | >=1.12.0 |
| `python-dotenv` | Load environment variables from .env | >=1.0.0 |

## Edit pyproject.toml

Open `pyproject.toml` and update the `[project]` section:

```toml
[project]
name = "deepagents-cli"
version = "0.1.0"
description = "Interactive AI coding assistant with tool calling and memory"
readme = "README.md"
requires-python = "<-3.11"
dependencies = [
    # LLM Framework
    "langchain>=1.2.13,<2.0.0",
    "langgraph>=1.1.2,<2.0.0",
    
    # OpenAI (GPT)
    "langchain-openai>=1.3.5,<2.0.0",
    "openai>=1.12.0,<2.0.0",
    
    # TUI Framework
    "textual>=8.0.0,<9.0.0",
    
    # Utilities
    "python-dotenv>=1.0.0,<2.0.0",
    "aiosqlite>=0.19.0,<1.0.0",
    "httpx>=0.28.1,<1.0.0",
]
```

## Add Entry Point

Update `[project.scripts]` to define our CLI command:

```toml
[project.scripts]
deepagents = "deepagents_cli:main"
```

## Why These Versions?

We pin major versions but allow minor/patch updates:

- `>=1.2.13,<2.0.0` — Get 1.x updates, avoid breaking 2.0
- `>=8.0.0,<9.0.0` — Textual 8.x is stable, 9.x might break

## Install Dependencies

```bash
uv sync
```

This creates `.venv` and installs all dependencies.

## Verify Installation

```bash
uv run python -c "
import langchain
import langgraph
import textual
import openai
print('langchain:', langchain.__version__)
print('langgraph:', langgraph.__version__)
print('textual:', textual.__version__)
print('openai:', openai.__version__)
"
```

Expected output:

```
langchain: 1.2.13
langgraph: 1.1.2
textual: 8.0.0
openai: 1.12.0
```

## LangChain Core Concepts

LangChain provides abstractions over LLM providers:

```python
from langchain_openai import ChatOpenAI

# Create a model instance
model = ChatOpenAI(model="gpt-4o")

# Invoke (blocking)
response = model.invoke("What is 2+2?")

# Stream (token by token)
for token in model.stream("What is 2+2?"):
    print(token.content, end="")
```

## LangGraph Core Concepts

LangGraph builds agents as state machines:

```python
from langgraph.graph import StateGraph, END

# Define state
class AgentState(TypedDict):
    messages: list

# Build graph
graph = StateGraph(AgentState)
graph.add_node("llm", llm_node)
graph.add_edge("llm", END)
app = graph.compile()
```

## Textual Core Concepts

Textual builds TUI apps with reactive widgets:

```python
from textual.app import App, ComposeResult
from textual.widgets import Static

class MyApp(App):
    def compose(self) -> ComposeResult:
        yield Static("Hello, Terminal!")

    async def on_mount(self):
        self.title = "My App"

if __name__ == "__main__":
    app = MyApp()
    app.run()
```

## API Key Security

Your API key is a **secret credential**. Never hardcode it or commit it to git.

### The DO's and DON'Ts

**DO:**
- ✅ Store in environment variables
- ✅ Use `.env` files (gitignored)
- ✅ Use secret managers (1Password, Bitwarden)
- ✅ Rotate keys regularly
- ✅ Set spending limits on your API account

**DON'T:**
- ❌ Hardcode in source code
- ❌ Commit to git
- ❌ Share in Slack/Discord
- ❌ Log or print keys

### Using .env Files (Recommended)

1. **Install python-dotenv** (already in dependencies)

2. **Create `.env` file**:
   ```bash
   # .env - This file should NEVER be committed to git
   OPENAI_API_KEY=sk-...
   OPENAI_BASE_URL=https://api.openai.com/v1  # Optional: for custom endpoints
   ```

3. **Add `.env` to `.gitignore`**:
   ```
   # .gitignore
   .env
   .env.local
   .env.*.local
   secrets/
   ```

4. **Load in your code**:
   ```python
   from dotenv import load_dotenv
   import os

   load_dotenv()  # Loads variables from .env

   api_key = os.environ.get("OPENAI_API_KEY")
   ```

### System Keyring (Most Secure)

For production or shared machines:

```bash
# Install keyring
pip install keyring

# Store key in OS keychain
keyring set deepagents openai_api_key
# Enter your API key when prompted
```

```python
import keyring
api_key = keyring.get_password("deepagents", "openai_api_key")
```

### Custom OpenAI-Compatible APIs

You can use any OpenAI-compatible API (OpenCode, Fireworks AI, Together AI, etc.):

```bash
# .env
OPENAI_API_KEY=your-api-key
OPENAI_BASE_URL=https://api.opencode.ai/v1
DEEPAGENTS_MODEL=openai:gpt-4o  # Format: provider:model
```

### Checking for Leaked Keys

Run this to verify no keys are in your code:

```bash
# Search for patterns that look like API keys
grep -r "sk-[a-zA-Z0-9]" src/

# Check git status for .env files
git status
# Ensure .env is not listed as "new file"
```

### What If You Accidentally Commit a Key?

1. **Rotate the key immediately** — Generate a new key in your provider dashboard
2. **Remove from git history** — Use `git filter-branch` or BFG Repo-Cleaner
3. **Check for unauthorized usage** — Review API usage logs
4. **Never reuse the leaked key** — Treat it as compromised

## Key Takeaways

- **pyproject.toml** declares dependencies with version constraints
- **uv sync** installs all dependencies
- **LangChain** abstracts LLM providers (OpenAI, Anthropic, etc.)
- **LangGraph** builds agents as state machines with nodes and edges
- **Textual** creates terminal UIs with reactive widgets
- **NEVER hardcode API keys** — Use environment variables or secret managers

## Next Section

[Build Entry Point](./section-04-entry-point.md) — Create main.py with argparse.
