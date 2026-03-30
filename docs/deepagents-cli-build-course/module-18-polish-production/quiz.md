# Module 18 Quiz

Test your understanding of Polish & Production.

## Question 1

What is the correct base class for all Deep Agents CLI exceptions?

A) `Exception`
B) `BaseException`
C) `DeepAgentsError`
D) `AgentError`

<details>
<summary>Answer</summary>

**C) DeepAgentsError**

All custom exceptions should inherit from `DeepAgentsError` to enable catching any CLI-specific exception with a single `except` clause.
</details>

---

## Question 2

Why use structured logging (like structlog) instead of standard logging?

A) It's faster
B) It produces machine-parseable output for log aggregators
C) It's required by Python
D) It automatically fixes bugs

<details>
<summary>Answer</summary>

**B) It produces machine-parseable output for log aggregators**

Structured logging outputs JSON that can be easily parsed by log aggregation tools (ELK, Datadog, etc.), making debugging in production much easier.
</details>

---

## Question 3

What pytest fixture creates a temporary directory for tests?

A) `tmp_file`
B) `tmp_path`
C) `temp_dir`
D) `test_path`

<details>
<summary>Answer</summary>

**B) tmp_path**

The `tmp_path` fixture provides a temporary directory that is automatically cleaned up after the test. It's ideal for file operation tests.
</details>

---

## Question 4

What is the difference between unit tests and integration tests?

A) Unit tests run faster
B) Integration tests use real or close-to-real dependencies
C) Unit tests don't use mocks
D) Both A and B

<details>
<summary>Answer</summary>

**D) Both A and B**

Unit tests are fast (no I/O, mocked dependencies) while integration tests verify how components work together using real or near-real dependencies.
</details>

---

## Question 5

In pytest, how do you mark an async test function?

A) `@pytest.mark.asyncio`
B) `@pytest.mark.async`
C) `async def test_` prefix automatically enables async
D) Use `pytest-asyncio` but no decorator needed with `asyncio_mode = "auto"`

<details>
<summary>Answer</summary>

**D) Use pytest-asyncio but no decorator needed with `asyncio_mode = "auto"`**

With `asyncio_mode = "auto"` in pyproject.toml, pytest-asyncio automatically discovers and runs async tests. No explicit decorator is needed.
</details>

---

## Question 6

What does `monkeypatch` do in pytest?

A) Mocks external API calls
B) Modifies environment variables or module attributes temporarily
C) Creates fake files
D) Skips slow tests

<details>
<summary>Answer</summary>

**B) Modifies environment variables or module attributes temporarily**

`monkeypatch` is a pytest fixture that temporarily modifies environment variables, sys.path, or object attributes during a test.
</details>

---

## Question 7

Which build backend is configured in pyproject.toml?

A) `setuptools`
B) `distutils`
C) `hatchling`
D) `poetry`

<details>
<summary>Answer</summary>

**C) hatchling**

The modern standard is hatchling (part of the Hatch project). It's specified in `[build-system]` with `requires = ["hatchling"]`.
</details>

---

## Question 8

What is the purpose of `twine`?

A) Build packages
B) Run tests
C) Securely upload packages to PyPI
D) Manage dependencies

<details>
<summary>Answer</summary>

**C) Securely upload packages to PyPI**

`twine` is the recommended tool for securely uploading packages to PyPI. It uses HTTPS and verifies package metadata.
</details>

---

## Question 9

In GitHub Actions, what secret stores your PyPI API token?

A) `PYPI_PASSWORD`
B) `PYPI_TOKEN`
C) `PYPI_API_TOKEN`
D) `API_KEY`

<details>
<summary>Answer</summary>

**C) PYPI_API_TOKEN`**

The workflow uses `${{ secrets.PYPI_API_TOKEN }}`. You configure this in repository Settings > Secrets and variables > Actions.
</details>

---

## Question 10

What triggers the release/publish workflow in the CI/CD setup?

A) Every push to main
B) Every pull request
C) Push of a version tag (e.g., `v0.1.0`)
D) Manual trigger only

<details>
<summary>Answer</summary>

**C) Push of a version tag (e.g., `v0.1.0`)**

The publish job has `if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/v')`, meaning it only runs when a `v*` tag is pushed.
</details>

---

## Question 11

What does `concurrency` in GitHub Actions prevent?

A) Slow builds
B) Multiple workflow runs for the same ref simultaneously
C) Failed builds
D) Expensive builds

<details>
<summary>Answer</summary>

**B) Multiple workflow runs for the same ref simultaneously**

The `concurrency` group cancels in-progress runs when a new run starts for the same ref, saving resources and avoiding confusion.
</details>

---

## Question 12

What is hatch-vcs used for?

A) Managing Python versions
B) Git-based automatic versioning
C) Creating virtual environments
D) Building containers

<details>
<summary>Answer</summary>

**B) Git-based automatic versioning**

hatch-vcs reads the version from git tags, so you don't have to manually update version numbers in multiple places.
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **Exception hierarchy** — Inherit from `DeepAgentsError`
- **structlog** — Structured logging with JSON output
- **pytest fixtures** — `tmp_path`, `monkeypatch`, custom fixtures
- **pytest-asyncio** — Async test support with `asyncio_mode = "auto"`
- **build + twine** — Modern packaging and publishing
- **GitHub Actions** — CI/CD with testing, linting, coverage, publishing

## Course Complete

Congratulations! You've completed all 18 modules of the Deep Agents CLI Build Course. Your CLI now has:

- Full agent architecture with LangGraph
- Tool system with file operations and shell execution
- Human-in-the-loop approval flows
- Session persistence with SQLite
- Skills system with SKILL.md loading
- Memory system with AGENTS.md
- Backend abstraction for local and sandbox execution
- MCP integration
- Subagent support
- Textual TUI with slash commands
- Comprehensive testing and CI/CD

Go build something amazing!
