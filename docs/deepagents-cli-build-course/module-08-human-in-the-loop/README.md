# Module 8: Human-in-the-Loop

Pause before dangerous operations, require approval.

## Learning Objectives

By the end of this module, you will:

- Understand why human-in-the-loop (HITL) is critical for security
- Implement the interrupt system for pausing agent actions
- Build approval prompts that display tool details for review
- Configure `interrupt_on` settings per tool
- Add the `--auto-approve` flag for CI/non-interactive use

## Estimated Time

~2 hours

## Sections

1. [Why Human-in-the-Loop?](./section-01-why-hitl.md) — Security metaphor: bouncer at club
2. [The Interrupt System](./section-02-interrupt-system.md) — LangGraph interrupts and decision handling
3. [Approval Prompts](./section-03-approval-prompts.md) — UI for reviewing tool calls
4. [Configuring interrupt_on](./section-04-interrupt-on-config.md) — Per-tool approval settings
5. [Auto-Approve for CI](./section-05-auto-approve.md) — `--auto-approve` flag and batch mode
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, your CLI will:

- Pause before executing shell commands (`rm`, `git push --force`)
- Pause before writing or editing files
- Show clear approval prompts with tool details
- Support `--auto-approve` for scripted/CI usage
- Allow toggling auto-approve at runtime with `Shift+Tab`

## Key Concepts

| Concept | Description |
|---------|-------------|
| `interrupt_on` | Configuration dict mapping tool names to approval requirements |
| `InterruptOnConfig` | Defines allowed decisions (`approve`/`reject`) and descriptions |
| `auto_approve` | Runtime flag to bypass all interrupts |
| Approval prompt | UI showing tool name, arguments, and risk assessment |

## Prerequisites

- Completed Module 7: Shell Execution
- Understanding of LangGraph state machines
- Familiarity with Textual widgets

## Next Module

[Module 9: Session & History](../module-09-session-history/README.md) — Thread IDs and SQLite persistence for conversation history.
