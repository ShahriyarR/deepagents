# Section 6: CI/CD Setup

Automate testing, linting, and publishing with GitHub Actions.

## GitHub Actions Overview

GitHub Actions run workflows on:
- Push to any branch
- Pull requests
- Scheduled times
- Manual triggers

## Create Workflow Directory

```bash
mkdir -p .github/workflows
```

## Main CI Workflow

```yaml
# .github/workflows/ci.yml

name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          enable-cache: true
      
      - name: Set up Python
        run: uv python install 3.11 3.12
      
      - name: Install dependencies
        run: uv sync --all-groups
      
      - name: Lint
        run: uv run ruff check deepagents_cli/
      
      - name: Type check
        run: uv run ty check deepagents_cli/
      
      - name: Test
        run: uv run pytest tests/ -v --cov=deepagents_cli --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.xml
          fail_ci_if_error: false

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v4
      
      - name: Set up Python
        run: uv python install 3.11
      
      - name: Install build
        run: uv sync --group build
      
      - name: Build package
        run: uv run build
      
      - name: Check dist contents
        run: |
          uv run python -c "
          import zipfile
          whl = 'dist/' + [f for f in __import__('os').listdir('dist') if f.endswith('.whl')][0]
          print('Package contents:')
          for name in zipfile.ZipFile(whl).namelist():
            print(f'  {name}')
          "

  publish:
    name: Publish
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/v')
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v4
      
      - name: Set up Python
        run: uv python install 3.11
      
      - name: Install build
        run: uv sync --group build
      
      - name: Build package
        run: uv run build
      
      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
        with:
          password: ${{ secrets.PYPI_API_TOKEN }}
```

## Add GitHub Secrets

For PyPI publishing:

1. Go to your repository Settings
2. Navigate to Secrets and variables > Actions
3. Add `PYPI_API_TOKEN` with your PyPI API token

Get PyPI tokens from:
- PyPI: https://pypi.org/manage/account/#api-tokens
- TestPyPI: https://test.pypi.org/manage/account/#api-tokens

## Release Workflow

Create a workflow for releases:

```yaml
# .github/workflows/release.yml

name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    name: Create Release
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Get version
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: dist/*
          generate_release_notes: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Pre-commit Hooks

Add pre-commit for local checks:

```bash
uv add --group dev pre-commit
```

Create `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
  
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.0.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

Install hooks:

```bash
uv run pre-commit install
```

## Codecov Integration

Add codecov for coverage tracking:

1. Go to https://codecov.io and add your repo
2. The workflow already includes `codecov/codecov-action@v4`

View coverage at `https://codecov.io/gh/<owner>/<repo>`

## Matrix Testing

Test across Python versions:

```yaml
jobs:
  test:
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      # ...
```

## Key Takeaways

- **GitHub Actions** — Free CI/CD for public repos
- **Concurrency groups** — Cancel outdated runs
- **Matrix strategy** — Test across Python versions
- **Secrets** — Store tokens securely
- **Pre-commit** — Local quality checks
- **Codecov** — Coverage tracking

## Next Section

[Quiz](./quiz.md) — Test your understanding of Module 18.
