# Section 4: Load into System Prompt

Injecting skills via progressive disclosure pattern.

## Progressive Disclosure

Skills use **progressive disclosure** to balance prompt size with rich content:

1. **Always visible** — Skill name, description, path
2. **On-demand** — Full SKILL.md content loaded only when needed

This keeps the system prompt manageable while providing detailed instructions when a skill is invoked.

## System Prompt Template

The skills section is injected into the system prompt:

```python
SKILLS_SYSTEM_PROMPT = """
## Skills System

You have access to a skills library that provides specialized capabilities.

{skills_locations}

**Available Skills:**

{skills_list}

**How to Use Skills (Progressive Disclosure):**

1. **Recognize when a skill applies**: Check if the task matches a skill's description
2. **Read the skill's full instructions**: Use the path shown above
3. **Follow the skill's instructions**: SKILL.md contains step-by-step workflows
4. **Access supporting files**: Skills may include helper scripts with absolute paths
"""
```

## Formatting Skills Locations

```python
def _format_skills_locations(self) -> str:
    """Format skills locations for display in system prompt."""
    locations = []
    
    for i, source_path in enumerate(self.sources):
        name = PurePosixPath(source_path.rstrip("/")).name.capitalize()
        suffix = " (higher priority)" if i == len(self.sources) - 1 else ""
        locations.append(f"**{name} Skills**: `{source_path}`{suffix}")
    
    return "\n".join(locations)
```

Example output:

```
**Built-in Skills**: `/skills/built-in/` (higher priority)
**User Skills**: `/skills/user/`
**Project Skills**: `/skills/project/`
```

## Formatting Skills List

```python
def _format_skills_list(self, skills: list[SkillMetadata]) -> str:
    """Format skills metadata for display in system prompt."""
    if not skills:
        paths = [f"{source_path}" for source_path in self.sources]
        return f"(No skills available yet. Add skills to {' or '.join(paths)})"
    
    lines = []
    for skill in skills:
        # Format: - **name**: description
        desc_line = f"- **{skill['name']}**: {skill['description']}"
        
        # Add optional annotations
        if skill.get("license") or skill.get("compatibility"):
            annotations = []
            if skill.get("license"):
                annotations.append(f"License: {skill['license']}")
            if skill.get("compatibility"):
                annotations.append(f"Compatibility: {skill['compatibility']}")
            desc_line += f" ({', '.join(annotations)})"
        
        lines.append(desc_line)
        
        # Add allowed tools if present
        if skill["allowed_tools"]:
            lines.append(f"  -> Allowed tools: {', '.join(skill['allowed_tools'])}")
        
        # Add path for full instructions
        lines.append(f"  -> Read `{skill['path']}` for full instructions")
    
    return "\n".join(lines)
```

Example output:

```
**Available Skills:**

- **web-research**: Structured approach to conducting thorough web research
  -> Allowed tools: browse, url_rewrite
  -> Read `skills/project/web-research/SKILL.md` for full instructions

- **code-review**: Systematic code review checklists and best practices
  -> Read `skills/user/code-review/SKILL.md` for full instructions
```

## Modifying the Request

`modify_request` injects the skills section into the system prompt:

```python
def modify_request(self, request: ModelRequest[ContextT]) -> ModelRequest[ContextT]:
    """Inject skills documentation into system message."""
    
    # Get skills metadata from state
    skills_metadata = request.state.get("skills_metadata", [])
    
    # Format locations and list
    skills_locations = self._format_skills_locations()
    skills_list = self._format_skills_list(skills_metadata)
    
    # Build skills section
    skills_section = self.system_prompt_template.format(
        skills_locations=skills_locations,
        skills_list=skills_list,
    )
    
    # Append to system message using helper utility
    new_system_message = append_to_system_message(
        request.system_message,
        skills_section,
    )
    
    return request.override(system_message=new_system_message)
```

## The `append_to_system_message` Helper

```python
from langchain_core.messages import SystemMessage

def append_to_system_message(
    existing: SystemMessage | str,
    additional: str,
) -> SystemMessage:
    """Append content to a system message."""
    if isinstance(existing, str):
        content = existing
    else:
        content = existing.content
    
    new_content = f"{content}\n{additional}"
    return SystemMessage(content=new_content)
```

## Wrap Model Call

The middleware wraps LLM calls:

```python
def wrap_model_call(
    self,
    request: ModelRequest[ContextT],
    handler: Callable[[ModelRequest[ContextT]], ModelResponse[ResponseT]],
) -> ModelResponse[ResponseT]:
    """Inject skills into system prompt and call LLM."""
    modified_request = self.modify_request(request)
    return handler(modified_request)
```

Async version:

```python
async def awrap_model_call(
    self,
    request: ModelRequest[ContextT],
    handler: Callable[[ModelRequest[ContextT]], Awaitable[ModelResponse[ResponseT]]],
) -> ModelResponse[ResponseT]:
    """Inject skills into system prompt and call LLM (async)."""
    modified_request = self.modify_request(request)
    return await handler(modified_request)
```

## Request Flow Diagram

```
1. before_agent (once per session)
   └── Loads skills_metadata into state

2. Agent decides to call LLM

3. wrap_model_call / awrap_model_call
   ├── modify_request(request)
   │   ├── Get skills_metadata from request.state
   │   ├── Format skills_locations and skills_list
   │   ├── Build skills_section from template
   │   └── Append to system_message
   │
   └── handler(modified_request) → LLM
```

## Full System Prompt Example

When skills are loaded, the system prompt includes:

```
## Skills System

You have access to a skills library that provides specialized capabilities.

**Built-in Skills**: `/skills/built-in/`
**User Skills**: `/skills/user/` (higher priority)

**Available Skills:**

- **web-research**: Structured approach to conducting thorough web research
  -> Allowed tools: browse, url_rewrite
  -> Read `skills/user/web-research/SKILL.md` for full instructions

- **code-review**: Systematic code review checklists and best practices
  -> Read `skills/user/code-review/SKILL.md` for full instructions

**How to Use Skills (Progressive Disclosure):**

1. Recognize when a skill applies
2. Read the skill's full instructions via the path shown
3. Follow the skill's step-by-step workflows
4. Access supporting files with absolute paths

When a user asks to research a topic, the agent recognizes this matches
"web-research" skill and reads the SKILL.md file to follow the structured
workflow.
```

## Key Takeaways

- **Progressive disclosure** — Metadata always visible, full content on demand
- **System prompt injection** — Skills section appended to existing system message
- **Request.override()** — Returns modified copy, original unchanged
- **Helper utilities** — `append_to_system_message` for clean code

## Next Section

[Skill Precedence](./section-05-skill-precedence.md) — Source ordering and override behavior.
