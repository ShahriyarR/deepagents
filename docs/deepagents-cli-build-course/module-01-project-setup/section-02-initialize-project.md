# Section 2: Initialize Project

Create the package structure using uv.

## What is uv?

**uv** is a fast, modern Python package manager written in Rust. It:

- Installs packages 10-100x faster than pip
- Manages virtual environments automatically
- Resolves dependencies correctly
- Replaces pip, pip-tools, pipx, and virtualenv

## Initialize the Project

```bash
# Create new project
uv init --name deepagents-cli

# Enter project directory
cd deepagents-cli
```

uv creates:

```
deepagents-cli/
├── pyproject.toml      # Package configuration
├── src/
│   └── deepagents_cli/
│       └── __init__.py
├── .python-version     # Python version lock
├── hello.py           # Sample script (delete later)
└── .venv/             # Virtual environment
```

## Delete the Sample File

Remove the auto-generated hello.py:

```bash
rm hello.py
```

## Verify the Structure

```bash
ls -la
```

You should see:

```
.pyproject.toml
.python-version
src/
```

## Understanding pyproject.toml

Open `pyproject.toml`. It looks like:

```toml
[project]
name = "deepagents-cli"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.11"
dependencies = []

[project.scripts]
deepagents-cli = "deepagents_cli:run"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

## Key pyproject.toml Sections

| Section | Purpose |
|---------|---------|
| `[project]` | Package metadata, dependencies |
| `[project.scripts]` | Console entry points |
| `[build-system]` | Build backend (hatchling, setuptools, etc.) |
| `[tool.hatch]` | Hatch-specific configuration |

## Add Package Directory Structure

Create the full package structure:

```bash
mkdir -p src/deepagents_cli/tools
mkdir -p src/deepagents_cli/middleware
mkdir -p src/deepagents_cli/ui
mkdir -p src/deepagents_cli/skills
```

## Create __init__.py Files

Each directory needs an `__init__.py` to be a Python package:

```bash
touch src/deepagents_cli/__init__.py
touch src/deepagents_cli/tools/__init__.py
touch src/deepagents_cli/middleware/__init__.py
touch src/deepagents_cli/ui/__init__.py
touch src/deepagents_cli/skills/__init__.py
```

## Final Structure

```
deepagents-cli/
├── pyproject.toml
├── .python-version
├── src/
│   └── deepagents_cli/
│       ├── __init__.py
│       ├── tools/
│       │   └── __init__.py
│       ├── middleware/
│       │   └── __init__.py
│       ├── ui/
│       │   └── __init__.py
│       └── skills/
│           └── __init__.py
└── .venv/
```

## Activate the Virtual Environment

uv automatically creates `.venv`. Activate it:

```bash
source .venv/bin/activate
```

Or use uv to run commands directly:

```bash
uv run python --version
```

## Test the Package

Run Python to verify the package is importable:

```bash
uv run python -c "import deepagents_cli; print('Import successful!')"
```

## Key Takeaways

- **uv init** creates a minimal Python project
- **pyproject.toml** defines package metadata and dependencies
- **src/** layout keeps source code separate from config
- **__init__.py** marks directories as Python packages
- **uv run** executes in the project's virtual environment

## Next Section

[Configure Dependencies](./section-03-dependencies.md) — Add LangChain, LangGraph, Textual, and Anthropic.
