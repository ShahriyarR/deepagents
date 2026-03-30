# Module 18: Polish & Production

Ship a robust, well-tested, release-ready CLI.

## Learning Objectives

By the end of this module, you will:

- Implement comprehensive error handling with custom exceptions
- Set up structured logging for debugging and production
- Write unit tests with pytest fixtures and mocks
- Write integration tests with pytest-asyncio
- Package the CLI for distribution using build
- Configure GitHub Actions CI/CD pipeline

## Prerequisites

- Modules 1-17 completed (full CLI built)
- pytest installed
- GitHub account for CI/CD

## Estimated Time

~4-5 hours

## Sections

1. [Error Handling](./section-01-error-handling.md) — Exception patterns, custom exceptions, graceful failures
2. [Logging & Debugging](./section-02-logging-debugging.md) — structlog, log levels, structured output
3. [Unit Tests](./section-03-unit-tests.md) — pytest fixtures, mocking, coverage
4. [Integration Tests](./section-04-integration-tests.md) — async tests, pytest-asyncio
5. [Packaging](./section-05-packaging.md) — build, wheel, sdist, publishing
6. [CI/CD Setup](./section-06-cicd-setup.md) — GitHub Actions, testing, linting, publishing
7. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, you'll have:

```
deepagents-cli/
├── pyproject.toml           # Includes test dependencies, build config
├── .github/
│   └── workflows/
│       └── ci.yml          # GitHub Actions pipeline
├── tests/
│   ├── unit_tests/
│   │   └── test_agent.py
│   └── integration_tests/
│       └── test_session.py
├── dist/
│   ├── deepagents_cli-0.1.0-py3-none-any.whl
│   └── deepagents_cli-0.1.0.tar.gz
└── Makefile                # test, lint, build, publish targets
```

## Key Concepts

- **Exception hierarchy** — Custom exceptions with context
- **structlog** — Structured logging for machines and humans
- **pytest fixtures** — Shared test setup with dependency injection
- **pytest-asyncio** — Async test support
- **build package** — Modern Python packaging
- **GitHub Actions** — Automated testing and publishing

## Next Module

Congratulations! You've completed the Deep Agents CLI Build Course. Your CLI is now production-ready with comprehensive tests, proper error handling, and automated CI/CD.
