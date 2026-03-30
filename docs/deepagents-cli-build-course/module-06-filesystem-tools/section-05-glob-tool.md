# Section 5: Glob Patterns

Pattern matching with Python's `glob` module.

## What are Glob Patterns?

Glob patterns are shell-style wildcards for matching file paths:

| Pattern | Meaning |
|---------|---------|
| `*.py` | All Python files in current directory |
| `**/*.py` | All Python files recursively |
| `src/*.py` | Python files in src directory |
| `**/test_*.py` | Files starting with `test_` ending in `.py` recursively |
| `[abc]*.txt` | Files starting with a, b, or c, ending in `.txt` |

## Using `glob.glob`

```python
import glob

files = glob.glob("*.py")  # All .py files in current directory
```

## Basic Glob Implementation

```python
import glob
from pathlib import Path


def glob_files(
    pattern: str,
    root_dir: str | None = None,
    recursive: bool = True,
) -> dict:
    """Find files matching a glob pattern.
    
    Args:
        pattern: Glob pattern to match (e.g., "**/*.py")
        root_dir: Root directory to search from (default: current directory)
        recursive: Whether to search recursively (default: True)
    
    Returns:
        Dictionary with matched file paths and metadata
    """
    try:
        if root_dir is None:
            search_root = Path.cwd()
        else:
            search_root = Path(root_dir).resolve()
            if not search_root.exists():
                return {
                    "success": False,
                    "error": f"Root directory not found: {root_dir}",
                    "matches": [],
                }
            if not search_root.is_dir():
                return {
                    "success": False,
                    "error": f"Root path is not a directory: {root_dir}",
                    "matches": [],
                }
        
        glob_pattern = str(search_root / pattern)
        
        matches = glob.glob(glob_pattern, recursive=recursive)
        
        results = []
        for match in matches:
            path = Path(match)
            if path.is_file():
                try:
                    stat = path.stat()
                    results.append({
                        "path": str(path),
                        "name": path.name,
                        "size": stat.st_size,
                        "modified": stat.st_mtime,
                    })
                except OSError:
                    results.append({
                        "path": str(path),
                        "name": path.name,
                        "size": None,
                        "modified": None,
                    })
        
        return {
            "success": True,
            "pattern": pattern,
            "root_dir": str(search_root),
            "matches": results,
            "count": len(results),
        }
    
    except Exception as e:
        return {
            "success": False,
            "error": f"Error searching files: {e!s}",
            "matches": [],
        }
```

## Common Patterns

```python
def find_python_files(root_dir: str | None = None) -> dict:
    """Find all Python files."""
    return glob_files("**/*.py", root_dir=root_dir)


def find_test_files(root_dir: str | None = None) -> dict:
    """Find all test files."""
    return glob_files("**/test_*.py", root_dir=root_dir) | glob_files("**/*_test.py", root_dir=root_dir)


def find_config_files(root_dir: str | None = None) -> dict:
    """Find configuration files."""
    return glob_files("**/*.{json,yaml,yml,toml}", root_dir=root_dir)


def find_markdown_files(root_dir: str | None = None) -> dict:
    """Find all Markdown files."""
    return glob_files("**/*.md", root_dir=root_dir) | glob_files("**/*.mdx", root_dir=root_dir)
```

## Filtering Matches

```python
def glob_files_filtered(
    pattern: str,
    root_dir: str | None = None,
    file_types: list[str] | None = None,
    min_size: int | None = None,
    max_size: int | None = None,
    exclude_patterns: list[str] | None = None,
) -> dict:
    """Find files matching a glob pattern with additional filters.
    
    Args:
        pattern: Glob pattern to match
        root_dir: Root directory to search from
        file_types: List of file extensions to include (e.g., [".py", ".md"])
        min_size: Minimum file size in bytes
        max_size: Maximum file size in bytes
        exclude_patterns: List of patterns to exclude
    
    Returns:
        Dictionary with filtered matches
    """
    result = glob_files(pattern, root_dir=root_dir)
    
    if not result["success"]:
        return result
    
    matches = result["matches"]
    
    if file_types:
        matches = [
            m for m in matches
            if any(m["path"].endswith(ext) for ext in file_types)
        ]
    
    if min_size is not None:
        matches = [m for m in matches if m["size"] is not None and m["size"] >= min_size]
    
    if max_size is not None:
        matches = [m for m in matches if m["size"] is not None and m["size"] <= max_size]
    
    if exclude_patterns:
        import fnmatch
        for exclude in exclude_patterns:
            matches = [m for m in matches if not fnmatch.fnmatch(m["name"], exclude)]
    
    result["matches"] = matches
    result["count"] = len(matches)
    
    return result
```

## Directory-Only Glob

```python
def glob_directories(
    pattern: str,
    root_dir: str | None = None,
) -> dict:
    """Find directories matching a glob pattern.
    
    Args:
        pattern: Glob pattern to match (e.g., "**/src")
        root_dir: Root directory to search from
    
    Returns:
        Dictionary with matched directories
    """
    try:
        if root_dir is None:
            search_root = Path.cwd()
        else:
            search_root = Path(root_dir).resolve()
        
        glob_pattern = str(search_root / pattern)
        matches = glob.glob(glob_pattern, recursive=True)
        
        results = []
        for match in matches:
            path = Path(match)
            if path.is_dir():
                results.append({
                    "path": str(path),
                    "name": path.name,
                })
        
        return {
            "success": True,
            "pattern": pattern,
            "root_dir": str(search_root),
            "matches": results,
            "count": len(results),
        }
    
    except Exception as e:
        return {
            "success": False,
            "error": f"Error searching directories: {e!s}",
            "matches": [],
        }
```

## Pathlib Alternative

For simpler cases, `pathlib` has built-in glob:

```python
from pathlib import Path

# All .py files recursively
py_files = list(Path(".").glob("**/*.py"))

# All files in src directory (non-recursive)
src_files = list(Path("src").glob("*"))

# Combine with filter
large_py_files = [
    f for f in Path(".").glob("**/*.py")
    if f.stat().st_size > 1000
]
```

## Testing the Tool

```python
# Find all Python files
result = glob_files("**/*.py")
print(f"Found {result['count']} Python files")

# Find test files only
result = glob_files("**/test_*.py")
print(f"Found {result['count']} test files")

# Find with size filter
result = glob_files_filtered(
    "**/*.py",
    min_size=1000,
    max_size=100000,
)
print(f"Found {result['count']} files between 1KB and 100KB")

# Find directories
result = glob_directories("**/node_modules")
print(f"Found {result['count']} node_modules directories")

# Exclude patterns
result = glob_files_filtered(
    "**/*.py",
    exclude_patterns=["*test*.py", "__pycache__/*"],
)
print(f"Found {result['count']} non-test Python files")
```

## Output Format

```json
{
  "success": true,
  "pattern": "**/*.py",
  "root_dir": "/home/user/project",
  "matches": [
    {
      "path": "/home/user/project/src/main.py",
      "name": "main.py",
      "size": 2048,
      "modified": 1699999999.0
    },
    {
      "path": "/home/user/project/src/utils.py",
      "name": "utils.py",
      "size": 1024,
      "modified": 1699999998.0
    }
  ],
  "count": 2
}
```

## Key Takeaways

- `glob.glob()` supports shell-style wildcards
- `**` matches any number of directories (recursive)
- `pathlib.Path.glob()` is an alternative using Path objects
- Filter results by size, type, or exclusion patterns
- `fnmatch` is useful for filtering with glob-like patterns

## Next Section

[Content Search](./section-06-grep-tool.md) — Regex search with `re.search`.
