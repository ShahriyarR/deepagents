# Prerequisites

Before starting this course, ensure you have the following:

## Required Knowledge

### Python Proficiency

You should be comfortable with:

- **Functions and classes** — Creating and using functions, defining classes
- **Imports** — Importing modules, understanding import paths
- **Type hints** — Using `str`, `int`, `list`, `dict`, `Optional`, etc.
- **Async/await** — Basic understanding of asynchronous programming
- **Context managers** — Using `with` statements

### Terminal Familiarity

You should know how to:

- Navigate directories with `cd`, `ls`, `pwd`
- Create files with `touch`, `mkdir`
- Run Python scripts with `python`
- Use environment variables
- Pipe commands together

## Required Software

### Python 3.11+

Install from [python.org](https://www.python.org/downloads/) or use pyenv:

```bash
# Using pyenv
pyenv install 3.11
pyenv global 3.11

# Verify
python --version
```

### uv

We use `uv` as our package manager. Install it:

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Or via pip
pip install uv

# Verify
uv --version
```

### Git

```bash
# macOS
brew install git

# Ubuntu/Debian
apt install git

# Verify
git --version
```

## API Keys

### OpenAI API Key (Default)

You'll need an OpenAI API key for GPT models (course default):

1. Sign up at [platform.openai.com](https://platform.openai.com/)
2. Navigate to API Keys
3. Create a new API key
4. **Save it securely** — This is a secret, like a password

#### Secure Setup (Required)

**NEVER hardcode your API key in source code or commit it to git.**

**Option 1: .env File (Recommended for Development)**

```bash
# Copy the example file
cp .env.example .env

# Edit .env and add your key
OPENAI_API_KEY=sk-...
```

Add `.env` to `.gitignore`:
```bash
echo ".env" >> .gitignore
```

**Option 2: Environment Variable**

```bash
# Add to your shell profile (~/.bashrc, ~/.zshrc, etc.)
export OPENAI_API_KEY="sk-..."

# Reload your shell
source ~/.bashrc  # or ~/.zshrc
```

### Alternative: OpenAI-Compatible APIs

You can use any OpenAI-compatible API instead:

**OpenCode:**
```bash
OPENAI_API_KEY=your-opencode-key
OPENAI_BASE_URL=https://api.opencode.ai/v1
```

**Fireworks AI:**
```bash
OPENAI_API_KEY=your-fireworks-key
OPENAI_BASE_URL=https://api.fireworks.ai/inference/v1
DEEPAGENTS_MODEL=openai:accounts/fireworks/models/llama-v3p1-70b-instruct
```

**Together AI:**
```bash
OPENAI_API_KEY=your-together-key
OPENAI_BASE_URL=https://api.together.xyz/v1
```

**Local/Custom Endpoint:**
```bash
OPENAI_API_KEY=not-needed-for-local
OPENAI_BASE_URL=http://localhost:8080/v1
```

### Security Checklist

Before proceeding, ensure:

- ✅ You have created a `.env` file (copied from `.env.example`)
- ✅ You have added `.env` to `.gitignore`
- ✅ Your API key is NOT hardcoded in any source files
- ✅ You have set spending limits on your API provider account

To verify:

```bash
# Check that .env is gitignored
git status
# Should NOT show .env as a new file

# Search for any hardcoded keys
grep -r "sk-" src/ || echo "No keys found in source"
```

## Nice to Have

### Text Editor

A good Python-aware text editor:

- **VS Code** with Python extension
- **PyCharm** (Community edition is free)
- **Neovim** with LSP

### Terminal

A modern terminal emulator:

- **iTerm2** (macOS)
- **Windows Terminal** (Windows)
- **Kitty** or **Alacritty** (cross-platform)

## Self-Assessment

Not sure if you're ready? Try these:

1. Can you explain what `async def` does?
2. Can you use `argparse` to parse `--input` and `positional` arguments?
3. Can you create a Python package with `__init__.py` files?

If you said yes to all three, you're ready!

If not, consider:
- [Python async tutorial](https://docs.python.org/3/library/asyncio.html)
- [argparse documentation](https://docs.python.org/3/library/argparse.html)
- [Python packaging guide](https://packaging.python.org/)

## Next Steps

Once you've confirmed your prerequisites:

1. [Quick Start](./quickstart.md) — Set up your development environment
2. [Module 1: Project Setup](../deepagents-cli-build-course/module-01-project-setup/README.md) — Start building
