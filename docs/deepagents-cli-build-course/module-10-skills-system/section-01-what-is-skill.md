# Section 1: What is a Skill?

Understanding the SKILL.md file format and skill directory structure.

## The Problem

Agents need specialized knowledge for domain-specific tasks:

- Research workflows for web searches
- Code review checklists and patterns
- Documentation writing standards
- Debugging systematic approaches

Hardcoding these into the agent makes it rigid. You need a **pluggable way** to extend agent capabilities.

## The Solution: Agent Skills

An agent skill is a **directory containing a SKILL.md file** that provides:

1. **Metadata** — Name, description, when to use
2. **Instructions** — Step-by-step workflows and best practices
3. **Supporting files** — Helper scripts, configs, reference docs

```
/skills/my-project/
├── SKILL.md          # Required: YAML frontmatter + markdown
└── scripts/          # Optional: helper scripts
    └── validate.py
```

## SKILL.md Format

SKILL.md uses YAML frontmatter for metadata, followed by markdown instructions:

```markdown
---
name: web-research
description: Structured approach to conducting thorough web research on any topic
license: MIT
compatibility: Python 3.10+
allowed_tools: browse url_rewrite
---

# Web Research Skill

## When to Use

Use this skill when:
- User asks to research a topic
- User wants to find information online
- User needs fact-checking on claims

## Research Workflow

1. **Search** — Use search tools to find relevant sources
2. **Evaluate** — Assess source credibility and relevance
3. **Organize** — Structure findings by theme or topic
4. **Synthesize** — Create coherent summary with citations

## Best Practices

- Always cite your sources
- Verify facts across multiple sources
- Note the date of information
- Distinguish between facts and opinions
```

## YAML Frontmatter Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Skill identifier (lowercase, alphanumeric, hyphens) |
| `description` | Yes | What the skill does (1-1024 chars) |
| `license` | No | License name for bundled code |
| `compatibility` | No | Environment requirements (1-500 chars) |
| `allowed_tools` | No | Space-delimited tool recommendations |
| `metadata` | No | Arbitrary key-value pairs |

### Name Constraints

Per the Agent Skills specification:

- 1-64 characters
- Lowercase alphanumeric and hyphens only
- Must not start or end with `-`
- Must not contain consecutive `--`
- Must match the parent directory name

```
Valid:   web-research, code-review, my-skill-2
Invalid: Web-Research, code_review, my--skill
```

## Skill Directory Structure

Each skill lives in its own directory:

```
skills/
├── web-research/          # Directory name = skill name
│   ├── SKILL.md          # Required
│   └── scripts/
│       └── scrape.py     # Optional supporting files
├── code-review/
│   ├── SKILL.md
│   └── checklists/
│       └── pr-checklist.md
└── documentation/
    └── SKILL.md
```

## Why Progressive Disclosure?

Skills follow **progressive disclosure**:

1. **Metadata visible always** — Name and description shown in system prompt
2. **Full content loaded on demand** — Read SKILL.md only when skill is invoked

This keeps the system prompt manageable while providing rich content when needed.

## Backend-Agnostic Storage

Skills work with any backend via the `BackendProtocol`:

- **FilesystemBackend** — Local directories (development)
- **StateBackend** — In-memory (testing)
- **Remote backends** — Cloud storage, databases (production)

```python
# Local filesystem
backend = FilesystemBackend(root_dir="/path/to/skills")

# In-memory (for testing)
backend = StateBackend(runtime_context)

# Remote (if implemented)
backend = S3Backend(bucket="my-skills")
```

## Loading Skills

The middleware discovers skills by:

1. **Listing directories** in the source path
2. **Checking for SKILL.md** in each directory
3. **Parsing YAML frontmatter** for metadata
4. **Storing metadata** in agent state

```python
# Discovery pseudocode
for skill_dir in backend.ls("/skills/"):
    if skill_dir.is_directory:
        skill_md_path = skill_dir / "SKILL.md"
        if skill_md_path.exists():
            metadata = parse_skill_metadata(skill_md_path)
            add_to_skills_list(metadata)
```

## Key Takeaways

- **Skill = directory + SKILL.md** — Self-contained capability modules
- **YAML frontmatter** — Structured metadata per Agent Skills spec
- **Markdown body** — Human-readable instructions and workflows
- **Progressive disclosure** — Metadata visible, full content loaded on demand
- **Backend abstraction** — Portable across storage types

## Next Section

[Skills Directory Scanner](./section-02-skills-directory-scanner.md) — Discover skills from backend storage using pathlib-style paths.
