# Section 5: Packaging

Prepare your CLI for distribution via PyPI or private indexes.

## Packaging Tools

Modern Python packaging uses:

- **build** — Build packages from source
- **twine** — Secure upload to PyPI
- **wheel** — Binary distribution format
- **sdist** — Source distribution

## Configure pyproject.toml

```toml
[project]
name = "deepagents-cli"
version = "0.1.0"
description = "Deep Agents CLI - Interactive AI coding assistant"
readme = "README.md"
requires-python = ">=3.11"
license = {text = "MIT"}
authors = [
    {name = "Your Name", email = "you@example.com"}
]
keywords = ["cli", "ai", "agent", "coding"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Environment :: Console",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]

dependencies = [
    "OpenAI>=0.25.0",
    "langchain>=0.2.0",
    "langgraph>=0.0.40",
    "textual>=0.50.0",
    "structlog>=24.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-mock>=3.12.0",
    "pytest-cov>=4.1.0",
    "ruff>=0.3.0",
    "ty>=0.1.0",
]
all-providers = [
    "langchain-openai>=0.1.0",
    "langchain-openai>=0.1.0",
]

[project.scripts]
deepagents = "deepagents_cli.main:main"

[build-system]
requires = ["hatchling", "hatch-vcs"]
build-backend = "hatchling.build"

[tool.hatch.version]
source = "vcs"
```

## Version Management with Hatch-VCS

Set up git-based versioning:

```bash
uv add --group build hatch-vcs hatchling
```

Initialize git if not done:

```bash
git init
git add .
git commit -m "Initial commit"
```

Tag your version:

```bash
git tag v0.1.0
```

Hatch will automatically:
- Read the version from git tags
- Include only tracked files in the package

## Create a pyproject.toml override

For git-based versioning, create `pyproject.toml` with:

```toml
[tool.hatch.version]
path = "deepagents_cli/__init__.py"
raw-options = [
    local_scheme = "no-local-version",
]

[tool.hatch.version.raw-options]
version = "0.1.0"  # Fallback version
```

Or use inline version:

```python
# deepagents_cli/__init__.py

__version__ = "0.1.0"
```

## Build the Package

Install build:

```bash
uv add --group build build
```

Build source and wheel:

```bash
uv run build
```

This creates:

```
dist/
├── deepagents_cli-0.1.0-py3-none-any.whl
└── deepagents_cli-0.1.0.tar.gz
```

## Verify the Build

Check package contents:

```bash
uv run pip install dist/deepagents_cli-*.whl --force-reinstall
deepagents --version
```

List contents:

```bash
uv run python -c "import zipfile; print(zipfile.ZipFile('dist/deepagents_cli-0.1.0-py3-none-any.whl').namelist())"
```

## Makefile for Common Tasks

Add to `Makefile`:

```makefile
.PHONY: build test lint format install publish clean

build: clean
	uv run build

test:
	uv run pytest tests/ -v --cov=deepagents_cli

lint:
	uv run ruff check deepagents_cli/

format:
	uv run ruff format deepagents_cli/

install: build
	uv pip install dist/deepagents_cli-*.whl --force-reinstall

publish: build
	uv run twine upload dist/*

clean:
	rm -rf dist/ build/ *.egg-info
```

## Publish to TestPyPI

Test your package before real release:

```bash
# Register at https://test.pypi.org

uv run twine upload --repository testpypi dist/*
```

Install from TestPyPI:

```bash
uv pip install --index-url https://test.pypi.org/simple/ deepagents-cli
```

## Publish to PyPI

```bash
# Set credentials
export TWINE_USERNAME=__token__
export TWINE_PASSWORD=pypi-xxxxxxxxxxxx

# Upload
uv run twine upload dist/*
```

## Key Takeaways

- **build package** — Modern build system
- **wheel + sdist** — Binary and source distributions
- **hatch-vcs** — Git-based versioning
- **twine** — Secure PyPI uploads
- **Makefile** — Convenient build commands
- **TestPyPI** — Test before publishing

## Next Section

[CI/CD Setup](./section-06-cicd-setup.md) — Automate testing and publishing with GitHub Actions.
