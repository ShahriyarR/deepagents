# Module 6: Filesystem Tools

Build comprehensive filesystem operations for the agent.

## Learning Objectives

By the end of this module, you will:

- Implement a directory listing tool (`ls`)
- Build a paginated file reading tool (`read_file`)
- Create a file writing tool (`write_file`)
- Implement in-place file editing (`edit_file`)
- Add glob-based pattern matching (`glob`)
- Build content search functionality (`grep`)

## Prerequisites

- Completed Module 3: Tool System
- Completed Module 4: Agent Architecture
- Understanding of Python `pathlib`, `os`, and `re` modules

## Estimated Time

~3-4 hours

## Sections

1. [Directory Listing](./section-01-ls-tool.md) — List files with `os.listdir`
2. [File Reading](./section-02-read-file.md) — Read with offset/limit pagination
3. [File Writing](./section-03-write-file.md) — Create and overwrite files
4. [File Editing](./section-04-edit-file.md) — In-place text replacement
5. [Glob Patterns](./section-05-glob-tool.md) — Pattern matching with `glob`
6. [Content Search](./section-06-grep-tool.md) — Regex search with `re.search`
7. [Quiz](./quiz.md) — Test your understanding

## What You'll Build

At the end of this module, you'll have a complete filesystem tool suite:

```python
from my_cli.tools import (
    list_directory,
    read_file,
    write_file,
    edit_file,
    glob_files,
    grep_search,
)

# List directory contents
result = list_directory(path="/path/to/dir", show_hidden=True)

# Read file with pagination
result = read_file(path="/path/to/file.py", offset=0, limit=100)

# Write new file
result = write_file(path="/path/to/new_file.py", content="# new file")

# Edit existing file
result = edit_file(
    path="/path/to/file.py",
    old_string="old text",
    new_string="new text",
)

# Find files by pattern
result = glob_files(pattern="**/*.py", root_dir="/path/to/search")

# Search file contents
result = grep_search(
    pattern=r"def\s+\w+\(",
    path="/path/to/file.py",
    case_sensitive=True,
)
```

## Key Concepts

| Concept | Purpose |
|---------|---------|
| `os.listdir` | List directory entries without recursion |
| `pathlib.Path` | Object-oriented filesystem paths |
| Offset/Limit | Pagination for large files |
| `difflib` | Compute differences between texts |
| `glob` patterns | Shell-style wildcard matching |
| `re.search` | Regular expression content search |

## Next Module

[Module 7: Shell Execution](../module-07-shell-execution/README.md) — Execute shell commands safely with allowlisting.
