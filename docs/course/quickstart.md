# Quick Start

Get your development environment set up and build your first CLI in 15 minutes.

## Step 1: Clone the Template

Create a new project using `uv`:

```bash
# Create new project
uv init --name deepagents-cli

# Enter directory
cd deepagents-cli
```

## Step 2: Configure pyproject.toml

Open `pyproject.toml` and add the core dependencies:

```toml
[project]
name = "deepagents-cli"
version = "0.1.0"
description = "Interactive AI coding assistant"
requires-python = ">=3.11"
dependencies = [
    "langchain>=1.2.13,<2.0.0",
    "langgraph>=1.1.2,<2.0.0",
    "langchain-openai>=1.3.5,<2.0.0",
    "openai>=1.12.0,<2.0.0",
    "textual>=8.0.0,<9.0.0",
    "python-dotenv>=1.0.0,<2.0.0",
    "aiosqlite>=0.19.0,<1.0.0",
    "httpx>=0.28.1,<1.0.0",
]

[project.scripts]
deepagents = "deepagents_cli:main"
```

## Step 3: Create Security Files

Before writing code, set up security to protect your API keys:

```bash
# Create .gitignore
cat > .gitignore << 'EOF'
# Environment variables (NEVER commit these)
.env
.env.local
.env.*.local

# Python
__pycache__/
*.py[cod]
*.egg-info/
.venv/

# IDE
.vscode/
.idea/
EOF
```

```bash
# Create .env.example (template for others)
cat > .env.example << 'EOF'
# Deep Agents CLI Configuration
# Copy this file to .env and add your actual API key

# OpenAI API Key (required)
# Get from: https://platform.openai.com/api-keys
OPENAI_API_KEY=sk-your-key-here

# Optional: Custom OpenAI-compatible API
# OPENAI_BASE_URL=https://api.openai.com/v1
# DEEPAGENTS_MODEL=openai:gpt-4o
EOF
```

## Step 4: Set Up Your API Key

**⚠️ SECURITY WARNING: Never commit API keys to git!**

```bash
# Copy the example file
cp .env.example .env

# Edit .env with your actual key (replace sk-your-key-here)
nano .env  # or vim, code, etc.
```

Your `.env` file should look like:
```
OPENAI_API_KEY=sk-abc123...
```

Verify `.env` is gitignored:
```bash
git status
# Should NOT show .env in the output
```

## Step 5: Install Dependencies

```bash
uv sync
```

## Step 6: Create Package Structure

```bash
mkdir -p src/deepagents_cli/{tools,middleware,ui,skills}
touch src/deepagents_cli/__init__.py
touch src/deepagents_cli/tools/__init__.py
touch src/deepagents_cli/middleware/__init__.py
touch src/deepagents_cli/ui/__init__.py
touch src/deepagents_cli/skills/__init__.py
```

## Step 7: Create Entry Point

Create `src/deepagents_cli/main.py`:

```python
"""Deep Agents CLI - Entry point."""

import argparse
import sys


def main() -> int:
    """Main entry point."""
    parser = argparse.ArgumentParser(description="Deep Agents CLI")
    parser.add_argument("--version", action="store_true")
    parser.add_argument("command", nargs="?", default="run")
    
    args = parser.parse_args()
    
    if args.version:
        print("deepagents-cli 0.1.0")
        return 0
    
    print(f"Starting Deep Agents CLI in {args.command} mode...")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Update `src/deepagents_cli/__init__.py`:

```python
"""Deep Agents CLI."""

__version__ = "0.1.0"

__all__ = ["main"]


def main() -> int:
    """Entry point."""
    from deepagents_cli.main import main as _main
    return _main()
```

## Step 8: Test It

```bash
uv run deepagents --version
```

Should output:

```
deepagents-cli 0.1.0
```

## What Just Happened?

You've created a working CLI package with:

- **uv** for dependency management
- **argparse** for CLI argument handling
- **Entry points** defined in pyproject.toml
- **.gitignore** to prevent committing secrets
- **.env.example** as a template for others

## Next Steps

Congratulations! You just completed **Module 0** (unwritten but implied).

Continue with:

- [Module 1: Project Setup](../deepagents-cli-build-course/module-01-project-setup/README.md) — Build the complete entry point
- [Module 2: REPL & Streaming](../deepagents-cli-build-course/module-02-repl-streaming/README.md) — Add streaming AI responses

## Troubleshooting

### "command not found: deepagents"

Run with `uv`:

```bash
uv run deepagents --version
```

Or reinstall in editable mode:

```bash
uv pip install -e .
```

### "No module named 'deepagents_cli'"

Make sure you're in the project root and the virtual environment is activated:

```bash
cd deepagents-cli
source .venv/bin/activate
```

### "API key not found"

Make sure you created the `.env` file and it's in your project root. Also verify you've loaded it:

```python
from dotenv import load_dotenv
load_dotenv()  # Add this at the start of your program
```

### "I accidentally committed my API key!"

1. **Rotate your API key immediately** — Generate a new one in your provider dashboard
2. **Remove from git history**:
   ```bash
   git filter-branch --force --index-filter \
   'git rm --cached --ignore-unmatch .env' \
   --prune-empty --tag-name-filter cat -- --all
   ```
3. **Force push** (if already pushed to remote):
   ```bash
   git push origin --force --all
   ```
