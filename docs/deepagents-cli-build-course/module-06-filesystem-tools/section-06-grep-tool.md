# Section 6: Content Search

Regex-based content search using `re.search`.

## What is Grep?

`grep` searches file contents for patterns. Our tool version supports:

- Regular expressions (regex)
- Case-sensitive and insensitive search
- Line numbers in results
- Context around matches (before/after lines)

## Basic Grep Implementation

```python
import re
from pathlib import Path


def grep_search(
    pattern: str,
    path: str | None = None,
    content: str | None = None,
    case_sensitive: bool = True,
    regex: bool = True,
    max_results: int = 100,
) -> dict:
    """Search file contents for a pattern.
    
    Either path OR content must be provided:
    - If path is provided, reads and searches the file
    - If content is provided directly, searches that content
    
    Args:
        pattern: Pattern to search for
        path: File path to search (mutually exclusive with content)
        content: Content string to search (mutually exclusive with path)
        case_sensitive: Whether search is case-sensitive
        regex: Treat pattern as regex (True) or literal string (False)
        max_results: Maximum number of matches to return
    
    Returns:
        Dictionary with search results and metadata
    """
    if path is None and content is None:
        return {
            "success": False,
            "error": "Either path or content must be provided",
            "matches": [],
        }
    
    if path is not None and content is not None:
        return {
            "success": False,
            "error": "Provide either path or content, not both",
            "matches": [],
        }
    
    try:
        if path is not None:
            file_path = Path(path).resolve()
            
            if not file_path.exists():
                return {
                    "success": False,
                    "error": f"File not found: {path}",
                    "matches": [],
                }
            
            if not file_path.is_file():
                return {
                    "success": False,
                    "error": f"Path is not a file: {path}",
                    "matches": [],
                }
            
            try:
                file_content = file_path.read_text(encoding="utf-8")
            except UnicodeDecodeError:
                return {
                    "success": False,
                    "error": f"File is not valid UTF-8: {path}",
                    "matches": [],
                }
        else:
            file_content = content
        
        if regex:
            flags = 0 if case_sensitive else re.IGNORECASE
            compiled_pattern = re.compile(pattern, flags)
        else:
            escaped = re.escape(pattern)
            flags = 0 if case_sensitive else re.IGNORECASE
            compiled_pattern = re.compile(escaped, flags)
        
        matches = []
        lines = file_content.splitlines()
        
        for line_num, line in enumerate(lines, start=1):
            if regex:
                match = compiled_pattern.search(line)
            else:
                match = compiled_pattern.search(line)
            
            if match:
                match_text = match.group(0)
                start_pos = match.start()
                end_pos = match.end()
                
                matches.append({
                    "line_number": line_num,
                    "line": line,
                    "match": match_text,
                    "start": start_pos,
                    "end": end_pos,
                    "context": None,
                })
                
                if len(matches) >= max_results:
                    break
        
        return {
            "success": True,
            "pattern": pattern,
            "path": str(file_path) if path else None,
            "matches": matches,
            "count": len(matches),
            "total_lines_searched": len(lines),
            "case_sensitive": case_sensitive,
            "is_regex": regex,
            "truncated": len(matches) == max_results,
        }
    
    except re.error as e:
        return {
            "success": False,
            "error": f"Invalid regex pattern: {e!s}",
            "matches": [],
        }
    except PermissionError:
        return {
            "success": False,
            "error": f"Permission denied: {path}",
            "matches": [],
        }
    except Exception as e:
        return {
            "success": False,
            "error": f"Error searching file: {e!s}",
            "matches": [],
        }
```

## Search with Context

Add surrounding lines for context:

```python
def grep_search_with_context(
    pattern: str,
    path: str | None = None,
    content: str | None = None,
    case_sensitive: bool = True,
    regex: bool = True,
    context_before: int = 2,
    context_after: int = 2,
    max_results: int = 50,
) -> dict:
    """Search with surrounding context lines.
    
    Args:
        pattern: Pattern to search for
        path: File path to search
        content: Content string to search
        case_sensitive: Whether search is case-sensitive
        regex: Treat pattern as regex
        context_before: Number of lines before match to include
        context_after: Number of lines after match to include
        max_results: Maximum number of matches to return
    
    Returns:
        Dictionary with results including context
    """
    if path is None and content is None:
        return {
            "success": False,
            "error": "Either path or content must be provided",
            "matches": [],
        }
    
    try:
        if path is not None:
            file_path = Path(path).resolve()
            if file_path.exists():
                file_content = file_path.read_text(encoding="utf-8")
            else:
                return {
                    "success": False,
                    "error": f"File not found: {path}",
                    "matches": [],
                }
        else:
            file_content = content
        
        flags = 0 if case_sensitive else re.IGNORECASE
        compiled_pattern = re.compile(pattern, flags if regex else re.escape(pattern))
        
        matches = []
        lines = file_content.splitlines()
        
        for line_num, line in enumerate(lines, start=1):
            if compiled_pattern.search(line):
                start = max(0, line_num - context_before - 1)
                end = min(len(lines), line_num + context_after)
                
                context_lines = []
                for i in range(start, end):
                    context_lines.append({
                        "line_number": i + 1,
                        "line": lines[i],
                        "is_match": (i + 1) == line_num,
                    })
                
                match_obj = compiled_pattern.search(line)
                
                matches.append({
                    "line_number": line_num,
                    "line": line,
                    "match": match_obj.group(0) if match_obj else "",
                    "context": context_lines,
                })
                
                if len(matches) >= max_results:
                    break
        
        return {
            "success": True,
            "pattern": pattern,
            "path": str(file_path) if path else None,
            "matches": matches,
            "count": len(matches),
            "context_before": context_before,
            "context_after": context_after,
        }
    
    except re.error as e:
        return {
            "success": False,
            "error": f"Invalid regex pattern: {e!s}",
            "matches": [],
        }
    except Exception as e:
        return {
            "success": False,
            "error": f"Error searching: {e!s}",
            "matches": [],
        }
```

## Search Multiple Files

Search across many files:

```python
import glob


def grep_multi_file(
    pattern: str,
    file_pattern: str = "**/*",
    root_dir: str | None = None,
    case_sensitive: bool = True,
    regex: bool = True,
    max_files: int = 100,
    max_matches_per_file: int = 50,
) -> dict:
    """Search for pattern across multiple files.
    
    Args:
        pattern: Pattern to search for
        file_pattern: Glob pattern for files to search
        root_dir: Root directory to search from
        case_sensitive: Whether search is case-sensitive
        regex: Treat pattern as regex
        max_files: Maximum number of files to search
        max_matches_per_file: Maximum matches per file
    
    Returns:
        Dictionary with results per file
    """
    try:
        if root_dir is None:
            search_root = Path.cwd()
        else:
            search_root = Path(root_dir).resolve()
        
        glob_pattern = str(search_root / file_pattern)
        files = glob.glob(glob_pattern, recursive=True)
        files = [f for f in files if Path(f).is_file()][:max_files]
        
        flags = 0 if case_sensitive else re.IGNORECASE
        compiled_pattern = re.compile(pattern, flags if regex else re.escape(pattern))
        
        results_by_file = {}
        
        for file_path in files:
            try:
                content = Path(file_path).read_text(encoding="utf-8")
            except (UnicodeDecodeError, PermissionError):
                continue
            
            matches = []
            lines = content.splitlines()
            
            for line_num, line in enumerate(lines, start=1):
                if compiled_pattern.search(line):
                    match_obj = compiled_pattern.search(line)
                    matches.append({
                        "line_number": line_num,
                        "line": line.rstrip(),
                        "match": match_obj.group(0) if match_obj else "",
                    })
                    
                    if len(matches) >= max_matches_per_file:
                        break
            
            if matches:
                results_by_file[file_path] = {
                    "path": file_path,
                    "match_count": len(matches),
                    "matches": matches,
                }
        
        total_matches = sum(r["match_count"] for r in results_by_file.values())
        
        return {
            "success": True,
            "pattern": pattern,
            "root_dir": str(search_root),
            "file_pattern": file_pattern,
            "files_searched": len(files),
            "files_with_matches": len(results_by_file),
            "total_matches": total_matches,
            "results": list(results_by_file.values()),
        }
    
    except Exception as e:
        return {
            "success": False,
            "error": f"Error in multi-file search: {e!s}",
            "results": [],
        }
```

## Common Search Patterns

```python
def find_function_definitions(pattern: str = r"^def\s+\w+\(") -> dict:
    """Find function definitions."""
    return grep_search(pattern, regex=True)


def find_class_definitions(pattern: str = r"^class\s+\w+") -> dict:
    """Find class definitions."""
    return grep_search(pattern, regex=True)


def find_imports(pattern: str = r"^import\s+|^from\s+") -> dict:
    """Find import statements."""
    return grep_search(pattern, regex=True)


def find_comments(pattern: str, case_sensitive: bool = False) -> dict:
    """Find comments containing pattern."""
    return grep_search(
        pattern,
        regex=False,
        case_sensitive=case_sensitive,
    )


def find_urls(pattern: str = r"https?://[^\s]+") -> dict:
    """Find URLs in content."""
    return grep_search(pattern, regex=True)
```

## Testing the Tool

```python
# Find all function definitions
result = grep_search(r"def\s+\w+\(", path="src/main.py")
print(f"Found {result['count']} functions")

# Find with context
result = grep_search_with_context(
    r"class\s+\w+",
    path="src/main.py",
    context_before=2,
    context_after=2,
)
for match in result["matches"][:3]:
    print(f"Line {match['line_number']}: {match['line']}")

# Search multiple files
result = grep_multi_file(
    pattern=r"TODO|FIXME|XXX",
    file_pattern="**/*.py",
    root_dir="src",
)
print(f"Found {result['total_matches']} TODOs across {result['files_with_matches']} files")

# Case-insensitive search
result = grep_search("hello", path="README.md", case_sensitive=False, regex=False)
print(f"Found {result['count']} matches")
```

## Output Format

```json
{
  "success": true,
  "pattern": "def\\s+\\w+\\(",
  "path": "/home/user/project/src/main.py",
  "matches": [
    {
      "line_number": 10,
      "line": "def hello_world():",
      "match": "def hello_world(",
      "start": 0,
      "end": 16,
      "context": null
    }
  ],
  "count": 5,
  "total_lines_searched": 100,
  "case_sensitive": true,
  "is_regex": true,
  "truncated": false
}
```

With context:

```json
{
  "success": true,
  "pattern": "class\\s+\\w+",
  "path": "/home/user/project/src/main.py",
  "matches": [
    {
      "line_number": 5,
      "line": "class MyClass:",
      "match": "class MyClass",
      "context": [
        {"line_number": 3, "line": "# Module", "is_match": false},
        {"line_number": 4, "line": "", "is_match": false},
        {"line_number": 5, "line": "class MyClass:", "is_match": true},
        {"line_number": 6, "line": "    def __init__(self):", "is_match": false}
      ]
    }
  ],
  "count": 1,
  "context_before": 2,
  "context_after": 2
}
```

## Key Takeaways

- `re.search()` finds first match, `re.findall()` finds all
- Compile patterns with `re.compile()` for repeated use
- Use `re.IGNORECASE` flag for case-insensitive search
- Always escape user input with `re.escape()` for literal search
- Context lines help users understand where matches occur

## Next Section

[Quiz](./quiz.md) — Test your understanding of filesystem tools.
