# Module 10: Skills System

Load custom instructions from SKILL.md files to extend agent capabilities.

## Learning Objectives

By the end of this module, you will:

- Understand the SKILL.md file format and YAML frontmatter structure
- Learn how the skills directory scanner discovers skills via backends
- Build `SkillsMiddleware` to inject skills into the system prompt
- Understand skill precedence and how later sources override earlier ones
- Implement progressive disclosure for skill documentation

## Prerequisites

- Module 5 completed (Middleware Pipeline)
- Understanding of Python TypedDict and YAML parsing
- Familiarity with pathlib and backend protocols
- Basic understanding of agent system prompts

## Estimated Time

~3-4 hours

## Sections

1. [What is a Skill?](./section-01-what-is-skill.md) — SKILL.md format and structure
2. [Skills Directory Scanner](./section-02-skills-directory-scanner.md) — Backend-based skill discovery
3. [SkillsMiddleware](./section-03-skills-middleware.md) — Loading skills into the agent
4. [Load into System Prompt](./section-04-load-into-prompt.md) — Progressive disclosure pattern
5. [Skill Precedence](./section-05-skill-precedence.md) — Source ordering and overrides
6. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, you'll have a skills loading system that:

```python
from deepagents.backends.filesystem import FilesystemBackend
from deepagents.middleware.skills import SkillsMiddleware

backend = FilesystemBackend(root_dir="/path/to/skills")
middleware = SkillsMiddleware(
    backend=backend,
    sources=[
        "/skills/user/",      # Lower priority
        "/skills/project/",    # Higher priority (overrides user)
    ],
)

# Skills are discovered from directories containing SKILL.md
# and injected into the system prompt via progressive disclosure
```

## Key Concepts

- **SKILL.md format** — YAML frontmatter + markdown instructions
- **SkillMetadata** — TypedDict with name, description, path, license
- **Backend abstraction** — Portable skill discovery across storage types
- **Progressive disclosure** — Show metadata first, full content on demand
- **Last-one-wins** — Later sources override earlier ones

## Architecture Preview

```
┌─────────────────────────────────────────────────────────────────┐
│                      SkillsMiddleware                            │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │  /skills/    │    │  /skills/    │    │  /skills/    │     │
│  │  built-in/   │───▶│  user/       │───▶│  project/    │     │
│  │              │    │              │    │              │     │
│  └──────────────┘    └──────────────┘    └──────────────┘     │
│        │                  │                  │                    │
│        ▼                  ▼                  ▼                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Skill Discovery: ls() → download SKILL.md → parse YAML  ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  System Prompt Injection (Progressive Disclosure)          ││
│  │  - Name + Description visible in list                      ││
│  │  - Full SKILL.md loaded only when skill is invoked         ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

## Real-World Examples

Skills enable powerful agent extensions:

- **web-research** — Structured research workflows with search, organize, synthesize
- **code-review** — Pull request review checklists and patterns
- **documentation** — Writing guides following project standards
- **debugging** — Systematic troubleshooting workflows

## Next Module

[Module 11: Memory System](../module-11-memory-system/README.md) — Load context from AGENTS.md files with MemoryMiddleware.
