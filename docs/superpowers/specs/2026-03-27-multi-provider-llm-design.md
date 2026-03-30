# Design: Multi-Provider LLM Support

## Goal
Replace the course's fixed Anthropic provider with a configurable multi-provider approach that supports:
- OpenAI (GPT-4o, GPT-4, GPT-3.5-turbo)
- OpenAI-compatible APIs (OpenCode, Fireworks AI, Together AI, etc.)
- Easy switching between providers
- Configuration via environment variables
- **Security-focused API key management**

## Changes Required

### 1. Config Module (`config.py`)
- Add `openai_api_key` property
- Add `openai_base_url` property (for custom endpoints)
- Add `model_provider` property
- Add `default_model` with provider-specific defaults

### 2. Models Module (`models.py`)
- Create `create_model()` factory function
- Support `langchain-openai` package
- Detect provider from model name prefix (e.g., "openai:gpt-4")
- Support custom base_url for OpenAI-compatible APIs

### 3. pyproject.toml Updates
- Replace `langchain-anthropic` with `langchain-openai` or add both
- Add `openai` package
- Optional: Make providers optional dependencies

### 4. Course Content Updates

#### Module 2 (REPL & Streaming)
- Update to use OpenAI instead of Anthropic
- Show streaming with OpenAI
- Mention configurable providers
- **Add security section for API key management**

#### Module 3-18
- Update all code examples to use the configurable provider
- Add "Changing the Model" section

### 5. Environment Setup
Students should set:
```bash
# OpenAI
OPENAI_API_KEY=sk-...

# Or custom OpenAI-compatible API
OPENAI_API_KEY=your-api-key
OPENAI_BASE_URL=https://api.opencode.ai/v1

# Model selection
DEEPAGENTS_MODEL=openai:gpt-4o
```

## Provider Format

Use format: `<provider>:<model>`

Examples:
- `openai:gpt-4o`
- `anthropic:claude-3-sonnet`
- `fireworks:accounts/fireworks/models/llama-v3p1-70b-instruct`

## Implementation Details

```python
# config.py
@property
def llm_config(self) -> dict:
    """Get LLM configuration."""
    model_str = os.environ.get("DEEPAGENTS_MODEL", "openai:gpt-4o")
    provider, model = model_str.split(":", 1)
    
    return {
        "provider": provider,
        "model": model,
        "api_key": os.environ.get(f"{provider.upper()}_API_KEY"),
        "base_url": os.environ.get(f"{provider.upper()}_BASE_URL"),
    }

# models.py  
def create_model(config: dict = None):
    """Create LLM based on configuration."""
    cfg = config or get_config().llm_config
    
    if cfg["provider"] == "openai":
        from langchain_openai import ChatOpenAI
        return ChatOpenAI(
            model=cfg["model"],
            api_key=cfg["api_key"],
            base_url=cfg.get("base_url"),  # For custom endpoints
        )
    elif cfg["provider"] == "anthropic":
        from langchain_anthropic import ChatAnthropic
        return ChatAnthropic(
            model=cfg["model"],
            api_key=cfg["api_key"],
        )
    else:
        raise ValueError(f"Unknown provider: {cfg['provider']}")
```

## Dependencies

```toml
dependencies = [
    "langchain>=1.2.13,<2.0.0",
    "langgraph>=1.1.2,<2.0.0",
    "langchain-openai>=1.3.5,<2.0.0",  # OpenAI support
    # Optional: "langchain-anthropic>=1.3.5,<2.0.0",  # Anthropic support
    "textual>=8.0.0,<9.0.0",
    "aiosqlite>=0.19.0,<1.0.0",
    "httpx>=0.28.1,<1.0.0",
]
```

## Security-Focused API Key Management

### The Problem
API keys are secrets. If exposed:
- Someone can use your credits
- Attackers can access your data
- Keys can be leaked in git history
- Keys can be exposed in logs

### Security Rules

**DO:**
- ✅ Use environment variables
- ✅ Use `.env` files (gitignored)
- ✅ Use secret managers (1Password, Bitwarden, etc.)
- ✅ Rotate keys regularly
- ✅ Use separate keys for dev/staging/prod
- ✅ Set spending limits on your API account

**DON'T:**
- ❌ Hardcode keys in source code
- ❌ Commit keys to git
- ❌ Share keys in Slack/Discord
- ❌ Log keys or print them
- ❌ Use production keys for development

### Implementation

#### Option A: Environment Variables (Recommended for Course)

```bash
# Add to ~/.bashrc or ~/.zshrc
export OPENAI_API_KEY="sk-..."

# Or create a .env file in project root
echo "OPENAI_API_KEY=sk-..." > .env
```

```python
# Load from .env file (add python-dotenv to dependencies)
from dotenv import load_dotenv
load_dotenv()  # Loads from .env file

import os
api_key = os.environ.get("OPENAI_API_KEY")
```

**Why:** 
- Keys not in code
- .env files can be gitignored
- Easy to switch between projects

**Setup:**
1. Add `python-dotenv` to dependencies
2. Create `.env` file
3. Add `.env` to `.gitignore`
4. Load at startup

#### Option B: System Keyring (Most Secure)

```bash
# Install keyring
pip install keyring

# Store key securely
keyring set deepagents openai_api_key
# Enter your API key when prompted
```

```python
import keyring
api_key = keyring.get_password("deepagents", "openai_api_key")
```

**Why:** 
- Keys stored in OS keychain
- Encrypted at rest
- Works across reboots

#### Option C: Secret Managers (Production)

For production deployment:
- **1Password**: Use `op` CLI to inject secrets
- **Bitwarden**: Use `bw` CLI
- **AWS/GCP/Azure**: Use their secret manager services
- **Doppler**: Cloud secret manager

Example with 1Password:
```bash
op run --env-file=.env -- uv run python main.py
```

### Course Security Section

Add to Module 1 or Module 2:

```markdown
## API Key Security

Your API key is a secret credential. Treat it like a password.

### Never Hardcode

```python
# ❌ BAD - Key in code
deepagents = DeepAgents(api_key="sk-abc123...")

# ✅ GOOD - Key from environment
deepagents = DeepAgents(api_key=os.environ.get("OPENAI_API_KEY"))
```

### Never Commit to Git

Create `.gitignore`:
```
.env
*.env
secrets/
```

Check with:
```bash
git status
# Ensure .env is not listed as "new file"
```

### Using .env Files

1. Install python-dotenv:
   ```bash
   uv add python-dotenv
   ```

2. Create `.env`:
   ```
   OPENAI_API_KEY=sk-...
   ```

3. Load in your code:
   ```python
   from dotenv import load_dotenv
   load_dotenv()
   ```

4. Add to `.gitignore`:
   ```
   .env
   ```

### Verifying Security

Run this to check for leaked keys:
```bash
# Search for patterns that look like API keys
grep -r "sk-[a-zA-Z0-9]" src/

# Use git-secrets to prevent commits
git secrets --scan
```

If you accidentally commit a key:
1. Rotate the key immediately
2. Remove from git history (git filter-branch)
3. Check if key was used by attackers
```

## Benefits
1. Students can use any OpenAI-compatible API
2. Flexibility for different budgets/needs
3. Encourages understanding of LLM abstraction layer
4. Real-world pattern used in production
5. **Teaches security best practices**

## Trade-offs
1. More complex setup (but just environment variables)
2. Multiple API keys to manage (students choose one)
3. Different models have different capabilities (need to document)
4. **Security requires discipline (but protects students)**

## Next Steps

After this design is approved:
1. Update Module 1 to use `langchain-openai`
2. Add security section to Module 1 or 2
3. Update all subsequent modules
4. Create `.env.example` template
5. Update `.gitignore` to exclude `.env`

