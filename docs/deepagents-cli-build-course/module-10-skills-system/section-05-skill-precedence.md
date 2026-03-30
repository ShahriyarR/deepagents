# Section 5: Skill Precedence

Understanding source ordering and override behavior.

## The Problem

With multiple skill sources, you need to answer:

1. **Which source takes priority?** — When skills have the same name
2. **How are conflicts resolved?** — Same skill in multiple sources
3. **What if a source is missing?** — Graceful handling of absent directories

## The Solution: Last-One-Wins

Skills use **last-one-wins** deduplication:

- Sources are processed in order
- Skills with the same name are **overwritten** by later sources
- A skill can be **shadowed** by a higher-pcedence source

```python
sources = [
    "/skills/built-in/",   # Priority 0 (lowest)
    "/skills/user/",       # Priority 1
    "/skills/project/",    # Priority 2 (highest)
]

# Processing:
# 1. Load all skills from built-in
# 2. Load all skills from user, OVERWRITE if same name
# 3. Load all skills from project, OVERWRITE if same name
```

## Implementation

```python
def before_agent(self, state: SkillsState, runtime: Runtime, config: RunnableConfig) -> SkillsStateUpdate | None:
    """Load skills with last-one-wins precedence."""
    
    if "skills_metadata" in state:
        return None  # Already loaded
    
    backend = self._get_backend(state, runtime, config)
    all_skills: dict[str, SkillMetadata] = {}
    
    # Load from each source in order
    for source_path in self.sources:
        source_skills = _list_skills(backend, source_path)
        for skill in source_skills:
            # Later sources OVERWRITE earlier ones
            all_skills[skill["name"]] = skill
    
    skills = list(all_skills.values())
    return SkillsStateUpdate(skills_metadata=skills)
```

Using a dict ensures last-one-wins naturally:

```python
all_skills: dict[str, SkillMetadata] = {}

for skill in source_skills:
    all_skills[skill["name"]] = skill  # Overwrites if exists
```

## Precedence Levels

Typical source organization (lowest to highest):

| Priority | Source | Path | Purpose |
|----------|--------|------|---------|
| 0 | Built-in | `<package>/built_in_skills/` | Package-provided skills |
| 1 | User | `~/.deepagents/{agent}/skills/` | User's personal skills |
| 2 | User Alias | `~/.agents/skills/` | Claude Code compatibility |
| 3 | Project | `.deepagents/skills/` | Project-specific skills |
| 4 | Project Alias | `.agents/skills/` | Claude Code compatibility |
| 5 | Claude (experimental) | `~/.claude/skills/` | Claude official skills |
| 6 | Claude Project (experimental) | `.claude/skills/` | Claude project skills |

## Override Example

```
# /skills/built-in/web-research/SKILL.md
name: web-research
description: Basic research workflow

# /skills/project/web-research/SKILL.md  
name: web-research
description: Enhanced research with citation tracking
```

Result: Project skill **overrides** built-in skill because project has higher priority.

## Shadowing vs Merging

Skills are **shadowed** (overwritten), not **merged** (combined):

```python
# Source 1: web-research
name: web-research
description: Basic research workflow

# Source 2: web-research
allowed_tools: browse url_rewrite
```

If Source 2's skill only has `allowed_tools` in its frontmatter, the entire
skill from Source 1 is shadowed — you don't get both descriptions.

**Design implication:** Each skill source should have a complete, self-contained SKILL.md.

## Graceful Handling

Each source is individually try/except guarded:

```python
sources = [
    ("/skills/built-in/", "built-in"),
    ("/skills/user/", "user"),
    ("/skills/project/", "project"),
]

for source_path, source_label in sources:
    if not source_path or not source_path.exists():
        continue  # Skip missing sources
    
    try:
        backend = FilesystemBackend(root_dir=str(source_path))
        skills = _list_skills(backend, ".")
        # ... process skills
    except (OSError, KeyError, TypeError):
        logger.warning("Could not load skills from %s", source_path, exc_info=True)
        continue  # Skip failed sources, continue with others
```

A missing or broken source doesn't block other sources.

## Source Priority in System Prompt

The system prompt shows source order with priority indication:

```python
def _format_skills_locations(self) -> str:
    """Format skills locations with priority."""
    locations = []
    
    for i, source_path in enumerate(self.sources):
        name = PurePosixPath(source_path.rstrip("/")).name.capitalize()
        suffix = " (higher priority)" if i == len(self.sources) - 1 else ""
        locations.append(f"**{name} Skills**: `{source_path}`{suffix}")
    
    return "\n".join(locations)
```

Example output:

```
**Built-in Skills**: `/skills/built-in/`
**User Skills**: `/skills/user/` (higher priority)
**Project Skills**: `/skills/project/` (higher priority)
```

## Why Last-One-Wins?

This pattern enables:

1. **Base skills** — Package-provided defaults
2. **Personal customization** — Override with user skills
3. **Project overrides** — Further customization per project
4. **Clean shadowing** — No merge conflicts or confusing combinations

Alternative patterns and why we don't use them:

| Pattern | Problem |
|---------|---------|
| Merge (combine all fields) | Complex, unpredictable results |
| First-one-wins | Hard to override defaults |
| Priority fields in YAML | Fragile, requires manual config |

## CLI Source Organization

The CLI uses this precedence for skill commands:

```python
sources = [
    (built_in_skills_dir, "built-in", False),
    (user_skills_dir, "user", False),
    (user_agent_skills_dir, "user", False),        # Alias
    (project_skills_dir, "project", False),
    (project_agent_skills_dir, "project", False),  # Alias
    (user_claude_skills_dir, "claude (experimental)", True),
    (project_claude_skills_dir, "claude (experimental)", True),
]
```

Note that `user_agent_skills_dir` and `project_agent_skills_dir` are aliases
with the **same priority** as their non-alias counterparts — they're just
different paths to similar user/project skills.

## Key Takeaways

- **Last-one-wins** — Later sources override earlier ones
- **Dict-based deduplication** — Natural overwrite behavior
- **Individual error handling** — One failed source doesn't block others
- **Complete skills** — Each source should have self-contained SKILL.md
- **Priority order** — built-in < user < project < claude

## Next Section

[Quiz](./quiz.md) — Test your understanding of Skills System concepts.
