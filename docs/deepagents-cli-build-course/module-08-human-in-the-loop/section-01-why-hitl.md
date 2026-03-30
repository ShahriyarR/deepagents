# Section 1: Why Human-in-the-Loop?

Security metaphor: a bouncer at a club.

## The Problem

An AI agent with shell access and file write permissions is powerful — and dangerous. Without safeguards, it could:

- Delete your entire home directory with `rm -rf ~`
- Overwrite system配置文件 with corrupted data
- Push code to production without review
- Execute malicious commands from prompt injection

The agent is fast, but it's not infallible. A single misconstrued instruction can cause irreversible damage.

## The Bouncer Metaphor

Think of human-in-the-loop as a **bouncer at an exclusive club**:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   "I'd like to execute: git push --force origin main"       │
│                                                             │
│                         ▼                                   │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                  THE BOUNCER                         │   │
│   │                                                      │   │
│   │   "Hold on. Let me check your ID."                  │   │
│   │   "This is a destructive action. Do you approve?"   │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                         ▼                                   │
│              [Approve] or [Reject]                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

The bouncer (human) doesn't trust anyone blindly. They verify identity, assess risk, and can turn away anyone — even someone who seems legitimate.

## What Gets Paused

Not every action needs approval. The rule of thumb:

| Action | Risk | Needs Approval? |
|--------|------|------------------|
| Reading files (`cat`, `ls`) | Low | No — read-only |
| Searching (`grep`, `find`) | Low | No — read-only |
| Writing files (`write_file`) | Medium | Yes — can overwrite |
| Editing files (`edit_file`) | Medium | Yes — modifies existing |
| Shell commands (`execute`) | High | Yes — side effects |
| Task delegation (`task`) | High | Yes — spawns agents |
| Subagent management | High | Yes — remote execution |

## Why Not Block Everything?

Approval fatigue is real. If the agent asks for approval on every `ls` and `cat`, users will:

1. Click approve reflexively
2. Disable approvals entirely
3. Stop using the tool

The goal is **targeted friction** — pause on dangerous actions, not benign ones.

## Security Layers

Human-in-the-loop is one layer in a defense-in-depth strategy:

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: Command Allowlisting                                │
│    Whitelist safe commands (ls, cat, git)                  │
│    Block dangerous ones (rm -rf, wget | bash)               │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ Layer 2: Human-in-the-Loop                                  │
│    Dangerous commands require approval                      │
│    User sees exactly what will run                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ Layer 3: Sandboxing                                          │
│    Remote execution in isolated environments                │
│    Cannot access host filesystem directly                   │
└─────────────────────────────────────────────────────────────┘
```

Each layer catches what the others miss.

## When to Skip Approval

Some scenarios warrant auto-approval:

1. **CI/CD pipelines** — No human present to approve
2. **Batch scripts** — Automated workflows with trusted commands
3. **Read-only sessions** — Only safe tools available
4. **Explicitly trusted operations** — User runs with `--auto-approve`

The `--auto-approve` flag exists for these cases. It's a conscious opt-in, not a default.

## Key Takeaways

- Human-in-the-loop prevents accidental or malicious damage
- The bouncer metaphor explains the mental model
- Only dangerous actions require approval
- Defense-in-depth uses multiple security layers
- Auto-approve is available for CI/automated scenarios

## Next Section

[The Interrupt System](./section-02-interrupt-system.md) — How LangGraph handles pauses and decisions.
