# Module 10 Quiz

Test your understanding of Skills System concepts.

## Question 1

What is the required file inside a skill directory?

A) `README.md`
B) `SKILL.md`
C) `metadata.json`
D) `skill.yaml`

<details>
<summary>Answer</summary>

**B) SKILL.md**

Each skill directory must contain a `SKILL.md` file with YAML frontmatter and markdown instructions.
</details>

---

## Question 2

What does the YAML frontmatter `name` field represent?

A) The filename of the SKILL.md
B) The skill directory name
C) The skill identifier used for deduplication
D) The author's name

<details>
<summary>Answer</summary>

**C) The skill identifier used for deduplication**

The `name` field is the skill identifier (1-64 chars, lowercase alphanumeric and hyphens) used for last-one-wins deduplication when multiple sources contain skills with the same name.
</details>

---

## Question 3

What is "progressive disclosure" in the context of skills?

A) Skills are loaded incrementally as the conversation progresses
B) Metadata is always visible; full content is loaded only when a skill is invoked
C) Skills are revealed one at a time based on user queries
D) YAML frontmatter is revealed before markdown content

<details>
<summary>Answer</summary>

**B) Metadata is always visible; full content is loaded only when a skill is invoked**

Progressive disclosure keeps the system prompt manageable by showing skill name and description in the list, while loading the full SKILL.md content only when the skill is actually used.
</details>

---

## Question 4

What is the maximum allowed size for a SKILL.md file?

A) 1 MB
B) 5 MB
C) 10 MB
D) 50 MB

<details>
<summary>Answer</summary>

**C) 10 MB**

`MAX_SKILL_FILE_SIZE = 10 * 1024 * 1024` (10 MB) to prevent DoS attacks via large skill files.
</details>

---

## Question 5

Given `sources=["/skills/base/", "/skills/user/"]`, what happens if both contain a skill named "web-research"?

A) Both are kept and shown separately
B) The base skill is kept (first-one-wins)
C) The user skill overrides the base skill (last-one-wins)
D) An error is raised

<details>
<summary>Answer</summary>

**C) The user skill overrides the base skill (last-one-wins)**

Skills use last-one-wins deduplication. The user skill overwrites the base skill because it appears later in the sources list.
</details>

---

## Question 6

What method in SkillsMiddleware loads skills into state?

A) `wrap_model_call()`
B) `before_agent()`
C) `modify_request()`
D) `load_skills()`

<details>
<summary>Answer</summary>

**B) `before_agent()`**

`before_agent()` (and its async counterpart `abefore_agent()`) loads skill metadata into the agent state once per session.
</details>

---

## Question 7

What does `modify_request()` do in SkillsMiddleware?

A) Downloads SKILL.md files from the backend
B) Parses YAML frontmatter from skill content
C) Injects skills section into the system prompt
D) Updates skills_metadata in agent state

<details>
<summary>Answer</summary>

**C) Injects skills section into the system prompt**

`modify_request()` gets skills metadata from state, formats it, and appends the skills section to the system message using `append_to_system_message()`.
</details>

---

## Question 8

Why does SkillsMiddleware support backend factory functions?

A) For faster backend initialization
B) To support StateBackend which requires runtime context
C) To enable backend subclassing
D) For lazy loading of backends

<details>
<summary>Answer</summary>

**B) To support StateBackend which requires runtime context**

StateBackend requires a runtime context to be instantiated. Using a factory function (`lambda rt: StateBackend(rt)`) allows each agent to get its own backend instance with proper context.
</details>

---

## Question 9

What happens if a skill source directory doesn't exist?

A) An error is raised
B) The middleware fails to initialize
C) The source is skipped, other sources continue
D) Empty skills are used as fallback

<details>
<summary>Answer</summary>

**C) The source is skipped, other sources continue**

Each source is individually try/except-guarded. A missing or inaccessible directory doesn't block other sources from being loaded.
</details>

---

## Question 10

What utility function appends content to a SystemMessage?

A) `append_message()`
B) `concat_system()`
C) `append_to_system_message()`
D) `modify_system()`

<details>
<summary>Answer</summary>

**C) `append_to_system_message()`**

From `deepagents.middleware._utils`, this helper creates a new `SystemMessage` with additional content appended.
</details>

---

## Question 11

What constraint applies to the skill `name` field per Agent Skills specification?

A) Must be valid UTF-8
B) Must match the parent directory name
C) Must be unique across all sources
D) Must be a valid Python identifier

<details>
<summary>Answer</summary>

**B) Must match the parent directory name**

Per the Agent Skills specification, the `name` field must match the parent directory name containing the SKILL.md file.
</details>

---

## Question 12

What is the purpose of `PrivateStateAttr` in SkillsState?

A) Marks skills_metadata as encrypted
B) Marks state as local to this agent (not propagated to parent)
C) Marks state as persistent across sessions
D) Marks state as read-only

<details>
<summary>Answer</summary>

**B) Marks state as local to this agent (not propagated to parent)**

`PrivateStateAttr` marks the `skills_metadata` field as local state that is not propagated to parent agents in nested graph scenarios.
</details>

---

## Question 13

Why do we use `PurePosixPath` for path operations?

A) Faster than pathlib.Path
B) Platform-independent path representation
C) Required by the backend protocol
D) Supports Windows drive letters

<details>
<summary>Answer</summary>

**B) Platform-independent path representation**

`PurePosixPath` uses forward slashes regardless of OS, ensuring consistent POSIX-style paths across all platforms for backend path operations.
</details>

---

## Summary

Key concepts covered:

- **SKILL.md format** — YAML frontmatter + markdown body
- **Progressive disclosure** — Metadata visible, full content on demand
- **SkillsMiddleware** — Discovers, loads, and injects skills
- **Last-one-wins** — Later sources override earlier ones
- **Backend abstraction** — Portable across storage types
- **State-based loading** — Skills loaded once per session

## Next Module

[Module 11: Memory System](../module-11-memory-system/README.md) — Load context from AGENTS.md files with MemoryMiddleware.
