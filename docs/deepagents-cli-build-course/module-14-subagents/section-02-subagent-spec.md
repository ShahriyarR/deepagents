# Section 2: SubAgent Specification

Defining subagents with TypedDict format.

## The SubAgent TypedDict

`SubAgent` is a TypedDict that specifies a subagent's configuration:

```python
from deepagents.middleware.subagents import SubAgent

subagent: SubAgent = {
    "name": "researcher",
    "description": "Research agent for deep analysis",
    "system_prompt": "You are a research agent. Use search and browse tools...",
    "model": "openai:gpt-4o",
    "tools": [search_tool, browse_tool],
}
```

## Required Fields

### `name`

Unique identifier for the subagent:

- Used by the main agent when calling the `task` tool
- Must be lowercase alphanumeric with hyphens
- Must be unique across all subagents

```python
"name": "researcher"
```

### `description`

What the subagent does:

- The main agent uses this to decide when to delegate
- Be specific and action-oriented
- Mention the domain or expertise area

```python
"description": "Research agent for deep analysis on complex topics"
```

### `system_prompt`

Instructions for the subagent:

- Define the subagent's role and behavior
- Include tool usage guidance
- Specify output format requirements

```python
"system_prompt": """You are a research agent specialized in deep analysis.

When researching:
1. Use search tools to find relevant sources
2. Evaluate source credibility
3. Synthesize findings into a concise report

Return a structured report with citations.
"""
```

## Optional Fields

### `tools`

Tools the subagent can use:

```python
"tools": [search_tool, browse_tool, read_tool]
```

If not specified, the subagent inherits tools from the main agent via `default_tools`.

### `model`

Override the main agent's model:

```python
"model": "openai:gpt-4o"
```

Use the format `'provider:model-name'`. The SDK resolves this via `init_chat_model`.

### `middleware`

Additional middleware for custom behavior:

```python
from deepagents.middleware.skills import SkillsMiddleware

"middleware": [
    SkillsMiddleware(
        backend=FilesystemBackend(),
        sources=["/skills/research/"],
    )
]
```

Middleware is applied after the default subagent middleware stack.

### `interrupt_on`

Configure human-in-the-loop for specific tools:

```python
"interrupt_on": {
    "execute": True,  # Require approval for shell execution
    "write": False,   # Auto-approve file writes
}
```

Requires a checkpointer on the main agent.

### `skills`

Skill source paths for SkillsMiddleware:

```python
"skills": ["/skills/user/", "/skills/project/"]
```

## CompiledSubAgent

For custom agent implementations, use `CompiledSubAgent`:

```python
from deepagents.middleware.subagents import CompiledSubAgent
from langchain.agents import create_agent

custom_agent = create_agent(model, system_prompt, tools)

compiled_subagent: CompiledSubAgent = {
    "name": "custom",
    "description": "Custom built agent",
    "runnable": custom_agent,
}
```

The runnable must return state with a `messages` key for result communication.

## Default Subagent Prompt

The SDK provides a default system prompt:

```python
DEFAULT_SUBAGENT_PROMPT = (
    "In order to complete the objective that the user asks of you, "
    "you have access to a number of standard tools."
)
```

You can override this per subagent or let it inherit.

## General Purpose Subagent

The SDK includes a built-in general-purpose subagent:

```python
from deepagents.middleware.subagents import GENERAL_PURPOSE_SUBAGENT

# Base spec (caller adds model, tools, middleware)
GENERAL_PURPOSE_SUBAGENT = {
    "name": "general-purpose",
    "description": "General-purpose agent for researching complex questions...",
    "system_prompt": DEFAULT_SUBAGENT_PROMPT,
}
```

Enable it via the deprecated API or explicitly add it to your subagents list.

## Full Example

```python
from deepagents.middleware.subagents import SubAgentMiddleware, SubAgent

subagents: list[SubAgent] = [
    {
        "name": "researcher",
        "description": "Research agent for deep analysis",
        "system_prompt": """You are a research agent specialized in thorough analysis.

Your workflow:
1. Search for relevant information
2. Evaluate source credibility
3. Synthesize findings into a structured report

Report format:
- Executive summary
- Key findings (bullet points)
- Citations (source URLs)
- Confidence level (high/medium/low)
""",
        "model": "openai:gpt-4o",
        "tools": [search_tool, browse_tool, read_tool],
        "skills": ["/skills/research/"],
    },
    {
        "name": "security-reviewer",
        "description": "Security-focused code review agent",
        "system_prompt": """You are a security expert reviewing code.

Check for:
- SQL injection vulnerabilities
- XSS vulnerabilities
- Authentication/authorization issues
- Data exposure risks

Return a finding report with severity (critical/high/medium/low).
""",
        "model": "OpenAI:gpt-4o",
        "tools": [read_tool, grep_tool],
        "middleware": [CustomRateLimitMiddleware()],
    },
]

middleware = SubAgentMiddleware(
    backend=my_backend,
    subagents=subagents,
)
```

## State Key Exclusion

When passing state to subagents, these keys are excluded:

```python
_EXCLUDED_STATE_KEYS = {
    "messages",          # Subagent gets fresh messages
    "todos",             # No shared todo list
    "structured_response",
    "skills_metadata",   # Subagent loads its own skills
    "memory_contents",   # Subagent loads its own memory
}
```

This prevents parent state from leaking into child agents.

## Key Takeaways

- **SubAgent TypedDict** — Declarative specification format
- **Required fields** — `name`, `description`, `system_prompt`
- **Optional fields** — `tools`, `model`, `middleware`, `interrupt_on`, `skills`
- **CompiledSubAgent** — For custom pre-built agent runnables
- **State isolation** — Parent state keys excluded from subagent context

## Next Section

[SubAgentMiddleware](./section-03-subagent-middleware.md) — Building the middleware that adds the task tool.

(End of file - total 133 lines)
