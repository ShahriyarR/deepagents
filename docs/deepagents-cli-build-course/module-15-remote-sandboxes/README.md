# Module 15: Remote Sandboxes

Run code execution in isolated cloud VMs with full GPU access and persistent environments.

## Learning Objectives

By the end of this module, you will:

- Understand the difference between local and remote code execution
- Implement the `SandboxBackendProtocol` for swappable execution backends
- Integrate LangSmith sandbox for cloud-based code execution
- Add Daytona managed dev environment support
- Add Modal serverless compute integration
- Use the `--sandbox` flag to toggle between execution modes

## Prerequisites

- Module 1: Project Setup (CLI structure)
- Module 7: Shell Execution (local execution basics)
- Module 12: Backend Abstraction (provider pattern)

## Estimated Time

~3-4 hours

## Sections

1. [Local vs Remote Execution](./section-01-local-vs-remote.md) — When and why to use remote sandboxes
2. [Sandbox Interface](./section-02-sandbox-interface.md) — `SandboxBackendProtocol` design
3. [LangSmith Sandbox](./section-03-langsmith-sandbox.md) — Cloud execution with LangSmith
4. [Daytona Integration](./section-04-daytona-integration.md) — Managed dev environments
5. [Modal Integration](./section-05-modal-integration.md) — Serverless compute
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

A flexible sandbox system that switches between execution modes:

```bash
# Local execution (default)
$ deepagents run
> Write a fast sorting algorithm
# Executes locally in subprocess

# Remote execution via --sandbox flag
$ deepagents run --sandbox langsmith
# Executes in LangSmith cloud sandbox

$ deepagents run --sandbox daytona
# Executes in Daytona managed environment

$ deepagents run --sandbox modal
# Executes via Modal serverless functions
```

## Architecture Preview

```
┌─────────────────────────────────────────────────────────────┐
│                     CLI Execution Layer                      │
│                    main.py (--sandbox flag)                  │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                   SandboxBackendProtocol                      │
│         execute(code, lang) -> ExecutionResult              │
└─────────────────────────────────────────────────────────────┘
                    │           │           │
          ┌────────┘           │           └────────┐
          ▼                     ▼                     ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│  LocalBackend   │   │  LangSmithBackend │   │   ModalBackend  │
│  (subprocess)   │   │  (cloud VM)       │   │  (serverless)   │
└─────────────────┘   └─────────────────┘   └─────────────────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                ▼
                    ┌─────────────────┐
                    │  DaytonaBackend │
                    │ (managed dev)   │
                    └─────────────────┘
```

## Key Concepts Introduced

- **SandboxBackendProtocol** — Abstract interface for execution backends
- **LangSmith sandbox** — Cloud VM with full environment
- **Daytona** — Managed development environments
- **Modal** — Serverless compute with GPU support
- **--sandbox flag** — CLI option to select execution backend

## Next Module

[Module 16: Textual TUI](../module-16-textual-tui/README.md) — Build the full terminal user interface.
