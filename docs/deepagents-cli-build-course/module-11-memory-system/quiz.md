# Module 11 Quiz

Test your understanding of the Memory System.

## Question 1

What is the primary purpose of AGENTS.md?

A) To define slash commands
B) To store persistent project context
C) To provide skill documentation
D) To configure middleware

<details>
<summary>Answer</summary>

**B) To store persistent project context**

AGENTS.md provides project-wide context like conventions, commands, and structure that persists across sessions. SKILL.md is for skill documentation (C). AGENTS.md doesn't define slash commands (A) or configure middleware (D).
</details>

---

## Question 2

What is the key difference between AGENTS.md and SKILL.md?

A) AGENTS.md is loaded once, SKILL.md is loaded on-demand
B) AGENTS.md is for Python projects, SKILL.md is for all projects
C) There is no difference
D) SKILL.md has higher priority

<details>
<summary>Answer</summary>

**A) AGENTS.md is loaded once, SKILL.md is loaded on-demand**

AGENTS.md memory is loaded at session start for persistent context. SKILL.md files are loaded when referenced by the user. This is the fundamental distinction between memory and skills.
</details>

---

## Question 3

What does the `FilesystemBackend` protocol define?

A) How to make LLM calls
B) How to read, write, and list files
C) How to parse markdown
D) How to route CLI commands

<details>
<summary>Answer</summary>

**B) How to read, write, and list files**

The FilesystemBackend is a protocol that abstracts filesystem operations, enabling different backends (local, sandbox, container) to work with the memory system.
</details>

---

## Question 4

Why does the memory system walk up the directory tree?

A) To find the correct Python interpreter
B) To find AGENTS.md in the project root from any subdirectory
C) To check for .env files
D) To locate the virtual environment

<details>
<summary>Answer</summary>

**B) To find AGENTS.md in the project root from any subdirectory**

When you're in a subdirectory like `src/module/`, the walk-up search finds `AGENTS.md` in the project root, ensuring memory is available regardless of where the user starts the CLI.
</details>

---

## Question 5

In MemoryMiddleware, what is the correct injection target?

A) HumanMessage
B) AIMessage
C) SystemMessage
D) ToolMessage

<details>
<summary>Answer</summary>

**C) SystemMessage**

Memory is injected into the SystemMessage because it contains the agent's base instructions. Modifying system-level context ensures the agent always has project memory awareness.
</details>

---

## Question 6

What happens if no AGENTS.md file is found?

A) The CLI crashes
B) An error is logged
C) Memory middleware skips injection gracefully
D) A default AGENTS.md is created

<details>
<summary>Answer</summary>

**C) Memory middleware skips injection gracefully**

The middleware checks if memory content exists before attempting injection. If no AGENTS.md is found, it simply passes the request through without modification.
</details>

---

## Question 7

Why is middleware ordering important for MemoryMiddleware?

A) Memory must load after skills
B) Memory should run early to enhance context for other middleware
C) Order doesn't matter
D) Memory must be the last middleware

<details>
<summary>Answer</summary>

**B) Memory should run early to enhance context for other middleware**

MemoryMiddleware has lower priority (higher number), so it runs early in the request chain. This ensures other middleware like SkillsMiddleware see the memory-enhanced system prompt.
</details>

---

## Question 8

What is token budget management in context injection?

A) Limiting the number of LLM calls
B) Truncating memory content to fit context limits
C) Compressing file contents
D) Encrypting sensitive data

<details>
<summary>Answer</summary>

**B) Truncating memory content to fit context limits**

LLMs have maximum context windows. Token budget management truncates AGENTS.md content if it exceeds a threshold, preventing context overflow while still providing useful information.
</details>

---

## Question 9

Which pattern does MemoryMiddleware use for loading?

A) Eager loading
B) Lazy loading
C) Preloading all files
D) Synchronous loading only

<details>
<summary>Answer</summary>

**B) Lazy loading**

Memory uses lazy loading — memory is loaded on first LLM call, not at startup. This improves CLI startup performance and avoids loading files unnecessarily.
</details>

---

## Question 10

What is the relationship between FilesystemBackend and MemoryMiddleware?

A) MemoryMiddleware uses FilesystemBackend to find and read AGENTS.md
B) FilesystemBackend wraps MemoryMiddleware
C) They are independent systems
D) MemoryMiddleware replaces FilesystemBackend

<details>
<summary>Answer</summary>

**A) MemoryMiddleware uses FilesystemBackend to find and read AGENTS.md**

FilesystemBackend is injected into MemoryMiddleware, providing an abstraction for file operations. This allows the memory system to work with local files, remote sandboxes, or any backend that implements the protocol.
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **AGENTS.md** — Persistent project context (memory)
- **SKILL.md** — On-demand skill documentation (skills)
- **FilesystemBackend** — Protocol for abstracting file operations
- **MemoryMiddleware** — Injects memory into system prompt
- **Walk-up search** — Finds AGENTS.md from any subdirectory
- **Token budget** — Prevents context overflow

## Next Module

[Module 12: Backend Abstraction](../module-12-backend-abstraction/README.md) — Abstract filesystem operations for local and sandboxed execution.
