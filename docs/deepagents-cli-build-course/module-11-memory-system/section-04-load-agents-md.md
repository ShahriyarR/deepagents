# Section 4: Load AGENTS.md

Implement robust loading and parsing of AGENTS.md files.

## AGENTS.md Format

AGENTS.md uses markdown with optional structured sections:

```markdown
# Project Context

## Environment
- Python 3.11+
- Node.js 20+

## Commands
- `make test` — Run tests
- `make lint` — Run linters

## Key Files
- `src/main.py` — Entry point
- `src/api/` — API handlers
```

## Loading Function

```python
from pathlib import Path
from typing import Optional

def load_agents_md(
    root_path: Path,
    backend: FilesystemBackend,
) -> Optional[str]:
    """Load AGENTS.md from root_path or nearest parent."""
    result = find_agents_md(root_path, backend)
    
    if result is None:
        return None
    
    path, content = result
    print(f"[memory] Loaded from {path}")
    return content
```

## Walk-Up Search Implementation

```python
def find_agents_md(
    start: Path,
    backend: FilesystemBackend,
) -> Optional[tuple[Path, str]]:
    """Find AGENTS.md by walking up directory tree."""
    current = start.resolve()

    while True:
        agents_path = current / "AGENTS.md"
        
        if backend.exists(agents_path):
            try:
                content = backend.read(agents_path)
                return (agents_path, content)
            except Exception as e:
                print(f"[memory] Warning: Could not read {agents_path}: {e}")

        parent = current.parent
        if parent == current:
            break
        current = parent

    return None
```

## Content Validation

Don't inject empty or invalid content:

```python
def is_valid_memory(content: str) -> bool:
    """Check if memory content is worth loading."""
    if not content or not content.strip():
        return False
    
    stripped = content.strip()
    
    # Too short to be meaningful
    if len(stripped) < 10:
        return False
    
    return True
```

## Section-Based Parsing

Optionally parse into structured sections:

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class MemorySection:
    title: str
    content: str

@dataclass 
class MemoryContext:
    raw: str
    sections: list[MemorySection]
    title: Optional[str] = None

def parse_agents_md(content: str) -> MemoryContext:
    """Parse AGENTS.md into structured context."""
    lines = content.split("\n")
    sections = []
    current_title = None
    current_content = []
    
    for line in lines:
        if line.startswith("## "):
            if current_title is not None:
                sections.append(MemorySection(
                    title=current_title,
                    content="\n".join(current_content).strip()
                ))
            current_title = line[3:].strip()
            current_content = []
        else:
            current_content.append(line)
    
    if current_title is not None:
        sections.append(MemorySection(
            title=current_title,
            content="\n".join(current_content).strip()
        ))
    
    title = sections[0].title if sections else None
    
    return MemoryContext(raw=content, sections=sections, title=title)
```

## Format for System Prompt

Format memory for clean injection:

```python
def format_memory_for_prompt(context: MemoryContext) -> str:
    """Format memory context for system prompt injection."""
    if not context.sections:
        return context.raw
    
    parts = []
    for section in context.sections:
        parts.append(f"### {section.title}")
        parts.append(section.content)
        parts.append("")
    
    return "\n".join(parts)
```

## Integration with Backend

Complete implementation:

```python
class MemoryLoader:
    """Handles loading and formatting of AGENTS.md memory."""

    def __init__(self, backend: FilesystemBackend):
        self.backend = backend
        self._cache: Optional[MemoryContext] = None

    def load(self, root_path: Path) -> Optional[str]:
        result = find_agents_md(root_path, self.backend)
        
        if result is None:
            return None
        
        path, raw_content = result
        
        if not is_valid_memory(raw_content):
            return None
        
        context = parse_agents_md(raw_content)
        self._cache = context
        
        return format_memory_for_prompt(context)

    def get_cached(self) -> Optional[str]:
        """Get memory without reloading from disk."""
        if self._cache is None:
            return None
        return format_memory_for_prompt(self._cache)
```

## Cache Invalidation

Memory is typically session-scoped:

```python
class MemoryLoader:
    def __init__(self, backend: FilesystemBackend):
        self.backend = backend
        self._cache: Optional[MemoryContext] = None

    def load(self, root_path: Path) -> Optional[str]:
        result = find_agents_md(root_path, self.backend)
        # ... load and parse ...
        self._cache = context
        return format_memory_for_prompt(context)

    def clear_cache(self):
        """Clear cached memory."""
        self._cache = None
```

## CLI Integration

Wire into CLI startup:

```python
def run_cli(backend: FilesystemBackend | None = None):
    backend = backend or LocalBackend()
    root_path = Path.cwd()
    
    loader = MemoryLoader(backend)
    memory_content = loader.load(root_path)
    
    if memory_content:
        print(f"[memory] Loaded {len(memory_content)} chars from AGENTS.md")
    else:
        print("[memory] No AGENTS.md found, starting without memory")
    
    app = DeepAgentsApp(
        backend=backend,
        memory_loader=loader.get_cached,
    )
```

## Key Takeaways

- **Walk-up search** finds AGENTS.md from any subdirectory
- **Validation** prevents empty content injection
- **Structured parsing** enables section-based handling
- **Caching** avoids repeated disk reads
- **Clean formatting** ensures proper system prompt integration

## Next Section

[Context Injection](./section-05-context-injection.md) — Deep dive into how memory modifies the system prompt.
