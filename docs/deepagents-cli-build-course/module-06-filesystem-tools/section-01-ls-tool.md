# Section 1: Directory Listing

List directory contents using `os.listdir`.

## What is Directory Listing?

Directory listing is the ability to see what files and folders exist in a location. The `ls` command in Unix-like systems does this.

```
/project/
├── src/
│   ├── main.py
│   └── utils.py
├── tests/
│   └── test_main.py
└── pyproject.toml
```

## Using `os.listdir`

The `os.listdir()` function returns a list of entries in a directory:

```python
import os

entries = os.listdir("/path/to/directory")
print(entries)
# ['src', 'tests', 'pyproject.toml']
```

## Basic Tool Implementation

Create `src/deepagents_cli/tools/filesystem.py`:

```python
"""Filesystem tools for the CLI agent."""

from __future__ import annotations

import os
from dataclasses import dataclass
from pathlib import Path
from typing import Literal


@dataclass
class DirectoryEntry:
    """A single directory entry with metadata."""

    name: str
    path: str
    is_file: bool
    is_dir: bool
    size: int | None = None


def list_directory(
    path: str = ".",
    show_hidden: bool = False,
) -> dict:
    """List directory contents.
    
    Args:
        path: Directory path to list (default: current directory)
        show_hidden: Whether to show hidden files (starting with .)
    
    Returns:
        Dictionary with entries list and metadata
    """
    try:
        abs_path = Path(path).resolve()
        
        if not abs_path.exists():
            return {
                "success": False,
                "error": f"Path does not exist: {path}",
                "entries": [],
            }
        
        if not abs_path.is_dir():
            return {
                "success": False,
                "error": f"Path is not a directory: {path}",
                "entries": [],
            }
        
        entries = os.listdir(str(abs_path))
        
        if not show_hidden:
            entries = [e for e in entries if not e.startswith(".")]
        
        results = []
        for name in sorted(entries):
            entry_path = abs_path / name
            try:
                stat = entry_path.stat()
                entry = DirectoryEntry(
                    name=name,
                    path=str(entry_path),
                    is_file=entry_path.is_file(),
                    is_dir=entry_path.is_dir(),
                    size=stat.st_size if entry_path.is_file() else None,
                )
            except OSError:
                entry = DirectoryEntry(
                    name=name,
                    path=str(entry_path),
                    is_file=False,
                    is_dir=False,
                    size=None,
                )
            results.append(entry)
        
        return {
            "success": True,
            "path": str(abs_path),
            "entries": [
                {
                    "name": e.name,
                    "path": e.path,
                    "is_file": e.is_file,
                    "is_dir": e.is_dir,
                    "size": e.size,
                }
                for e in results
            ],
            "total": len(results),
        }
    
    except PermissionError:
        return {
            "success": False,
            "error": f"Permission denied: {path}",
            "entries": [],
        }
    except Exception as e:
        return {
            "success": False,
            "error": f"Error listing directory: {e!s}",
            "entries": [],
        }
```

## Filtering by Type

Add helpers to filter by file or directory:

```python
def list_files(path: str = ".") -> dict:
    """List only files in a directory."""
    result = list_directory(path, show_hidden=True)
    if result["success"]:
        result["entries"] = [
            e for e in result["entries"] if e["is_file"]
        ]
        result["total"] = len(result["entries"])
    return result


def list_directories(path: str = ".") -> dict:
    """List only directories in a directory."""
    result = list_directory(path, show_hidden=True)
    if result["success"]:
        result["entries"] = [
            e for e in result["entries"] if e["is_dir"]
        ]
        result["total"] = len(result["entries"])
    return result
```

## Tree View (Recursive Listing)

For a deeper view, implement recursive listing:

```python
def list_directory_tree(
    path: str = ".",
    max_depth: int = 3,
    current_depth: int = 0,
) -> dict:
    """List directory contents recursively.
    
    Args:
        path: Directory path to list
        max_depth: Maximum recursion depth
        current_depth: Current depth (for internal use)
    
    Returns:
        Nested dictionary representing directory tree
    """
    if current_depth >= max_depth:
        return {"path": path, "entries": [], "truncated": True}
    
    result = list_directory(path, show_hidden=False)
    
    if not result["success"]:
        return result
    
    tree_entries = []
    for entry in result["entries"]:
        if entry["is_dir"]:
            subtree = list_directory_tree(
                entry["path"],
                max_depth=max_depth,
                current_depth=current_depth + 1,
            )
            tree_entries.append(subtree)
        else:
            tree_entries.append(entry)
    
    return {
        "path": result["path"],
        "entries": tree_entries,
        "total": len(tree_entries),
    }
```

## Testing the Tool

```python
# Test basic listing
result = list_directory(".")
print(f"Found {result['total']} entries")

# Test with hidden files
result = list_directory(".", show_hidden=True)
print(f"Found {result['total']} entries (including hidden)")

# Test file-only listing
result = list_files(".")
print(f"Found {len([e for e in result['entries'] if e['is_file']])} files")
```

## Output Format

```json
{
  "success": true,
  "path": "/home/user/project",
  "entries": [
    {
      "name": "src",
      "path": "/home/user/project/src",
      "is_file": false,
      "is_dir": true,
      "size": null
    },
    {
      "name": "pyproject.toml",
      "path": "/home/user/project/pyproject.toml",
      "is_file": true,
      "is_dir": false,
      "size": 256
    }
  ],
  "total": 5
}
```

## Key Takeaways

- `os.listdir()` returns names only, not full paths
- Use `pathlib.Path` for safe path manipulation
- Always handle `PermissionError` for protected directories
- Filter hidden files (starting with `.`) unless requested
- Sort entries alphabetically for consistent output

## Next Section

[File Reading](./section-02-read-file.md) — Read file contents with pagination.
