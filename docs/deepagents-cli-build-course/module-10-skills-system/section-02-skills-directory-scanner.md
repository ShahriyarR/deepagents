# Section 2: Skills Directory Scanner

Discovering skills from backend storage using pathlib-style paths.

## The Scanner Concept

Skills live in **sources** — paths in a backend where skills are organized. The scanner:

1. Lists directories in a source path
2. Filters for directories containing SKILL.md
3. Downloads and parses each SKILL.md
4. Returns skill metadata

```
source_path/
├── skill-name-1/
│   └── SKILL.md   → Discovered
├── skill-name-2/
│   └── SKILL.md   → Discovered
└── readme.txt     → Ignored (not a directory)
```

## Backend Protocol

The scanner uses `BackendProtocol` for storage abstraction:

```python
from deepagents.backends.protocol import BackendProtocol

class BackendProtocol:
    def ls(self, path: str) -> LsResult: ...
    def download_files(self, paths: list[str]) -> list[DownloadResult]: ...
    async def als(self, path: str) -> LsResult: ...
    async def adownload_files(self, paths: list[str]) -> list[DownloadResult]: ...
```

This enables the same scanner to work with:
- Local filesystem
- In-memory state
- Remote storage (S3, etc.)

## Implementation: `_list_skills`

```python
from pathlib import PurePosixPath
from deepagents.backends.protocol import LsResult

def _list_skills(backend: BackendProtocol, source_path: str) -> list[SkillMetadata]:
    """List all skills from a backend source."""
    skills: list[SkillMetadata] = []
    
    # 1. List directories in source
    ls_result = backend.ls(source_path)
    items = ls_result.entries if isinstance(ls_result, LsResult) else ls_result
    
    # 2. Find skill directories
    skill_dirs = [
        item["path"] 
        for item in items or []
        if item.get("is_dir")
    ]
    
    if not skill_dirs:
        return []
    
    # 3. Build SKILL.md paths
    skill_md_paths = [
        (skill_dir_path, str(PurePosixPath(skill_dir_path) / "SKILL.md"))
        for skill_dir_path in skill_dirs
    ]
    
    # 4. Download all SKILL.md files
    paths_to_download = [path for _, path in skill_md_paths]
    responses = backend.download_files(paths_to_download)
    
    # 5. Parse each SKILL.md
    for (skill_dir_path, skill_md_path), response in zip(skill_md_paths, responses, strict=True):
        if response.error or response.content is None:
            continue
        
        content = response.content.decode("utf-8")
        directory_name = PurePosixPath(skill_dir_path).name
        
        skill_metadata = _parse_skill_metadata(
            content=content,
            skill_path=skill_md_path,
            directory_name=directory_name,
        )
        if skill_metadata:
            skills.append(skill_metadata)
    
    return skills
```

## Using PurePosixPath

Paths use POSIX conventions for portability:

```python
from pathlib import PurePosixPath

# Always use forward slashes
skill_md_path = PurePosixPath(skill_dir_path) / "SKILL.md"
path_str = str(skill_md_path)  # "skills/my-skill/SKILL.md"
```

This ensures consistent paths regardless of operating system.

## Async Version

The async version uses `await` for backend operations:

```python
async def _alist_skills(backend: BackendProtocol, source_path: str) -> list[SkillMetadata]:
    """List all skills from a backend source (async)."""
    skills: list[SkillMetadata] = []
    
    ls_result = await backend.als(source_path)
    items = ls_result.entries if isinstance(ls_result, LsResult) else ls_result
    
    skill_dirs = [
        item["path"]
        for item in items or []
        if item.get("is_dir")
    ]
    
    if not skill_dirs:
        return []
    
    skill_md_paths = [
        (skill_dir_path, str(PurePosixPath(skill_dir_path) / "SKILL.md"))
        for skill_dir_path in skill_dirs
    ]
    
    paths_to_download = [path for _, path in skill_md_paths]
    responses = await backend.adownload_files(paths_to_download)
    
    for (skill_dir_path, skill_md_path), response in zip(skill_md_paths, responses, strict=True):
        if response.error or response.content is None:
            continue
        
        content = response.content.decode("utf-8")
        directory_name = PurePosixPath(skill_dir_path).name
        
        skill_metadata = _parse_skill_metadata(
            content=content,
            skill_path=skill_md_path,
            directory_name=directory_name,
        )
        if skill_metadata:
            skills.append(skill_metadata)
    
    return skills
```

## Parsing SKILL.md

The `_parse_skill_metadata` function extracts YAML frontmatter:

```python
import re
import yaml
from typing import TypedDict

MAX_SKILL_FILE_SIZE = 10 * 1024 * 1024  # 10MB limit

def _parse_skill_metadata(
    content: str,
    skill_path: str,
    directory_name: str,
) -> SkillMetadata | None:
    """Parse YAML frontmatter from SKILL.md content."""
    
    # Check file size
    if len(content) > MAX_SKILL_FILE_SIZE:
        logger.warning("Skipping %s: content too large", skill_path)
        return None
    
    # Extract YAML frontmatter between --- markers
    frontmatter_pattern = r"^---\s*\n(.*?)\n---\s*\n"
    match = re.match(frontmatter_pattern, content, re.DOTALL)
    
    if not match:
        logger.warning("Skipping %s: no valid YAML frontmatter", skill_path)
        return None
    
    frontmatter_str = match.group(1)
    
    try:
        frontmatter_data = yaml.safe_load(frontmatter_str)
    except yaml.YAMLError as e:
        logger.warning("Invalid YAML in %s: %s", skill_path, e)
        return None
    
    if not isinstance(frontmatter_data, dict):
        return None
    
    name = str(frontmatter_data.get("name", "")).strip()
    description = str(frontmatter_data.get("description", "")).strip()
    
    if not name or not description:
        logger.warning("Skipping %s: missing required fields", skill_path)
        return None
    
    return SkillMetadata(
        name=name,
        description=description,
        path=skill_path,
        metadata=frontmatter_data.get("metadata", {}),
        license=str(frontmatter_data.get("license", "")).strip() or None,
        compatibility=str(frontmatter_data.get("compatibility", "")).strip() or None,
        allowed_tools=parse_allowed_tools(frontmatter_data.get("allowed-tools")),
    )
```

## SkillMetadata TypedDict

```python
class SkillMetadata(TypedDict):
    """Metadata for a skill per Agent Skills specification."""
    
    path: str
    """Path to the SKILL.md file."""
    
    name: str
    """Skill identifier (1-64 chars, lowercase alphanumeric and hyphens)."""
    
    description: str
    """What the skill does (1-1024 chars)."""
    
    license: str | None
    """License name for bundled code."""
    
    compatibility: str | None
    """Environment requirements."""
    
    metadata: dict[str, str]
    """Arbitrary key-value pairs."""
    
    allowed_tools: list[str]
    """Tool names the skill recommends."""
```

## Source Organization

Sources are simply paths containing skill directories:

```python
sources = [
    "/skills/built-in/",   # Package-provided skills
    "/skills/user/",       # User's personal skills
    "/skills/project/",    # Project-specific skills
]
```

Each source is scanned independently, and results are merged.

## Key Takeaways

- **Backend abstraction** — Scanner works with any storage backend
- **PurePosixPath** — Platform-independent path handling
- **Batch downloads** — Efficient parallel SKILL.md fetching
- **Size limits** — 10MB max to prevent DoS
- **YAML parsing** — Safe frontmatter extraction with `yaml.safe_load`

## Next Section

[SkillsMiddleware](./section-03-skills-middleware.md) — Build middleware to load skills and manage state.
