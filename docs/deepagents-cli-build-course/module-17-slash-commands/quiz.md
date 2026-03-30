# Module 17 Quiz

Test your understanding of Slash Commands.

## Question 1

What is the correct bypass tier for a command that opens a modal dialog?

A) `BypassTier.ALWAYS`
B) `BypassTier.CONNECTING`
C) `BypassTier.IMMEDIATE_UI`
D) `BypassTier.SIDE_EFFECT_FREE`
E) `BypassTier.QUEUED`

<details>
<summary>Answer</summary>

**C) `BypassTier.IMMEDIATE_UI`**

`IMMEDIATE_UI` is for commands that open modal UI immediately, deferring real work via `_defer_action`. Examples: `/model`, `/threads`, `/theme`.
</details>

---

## Question 2

Which field in `SlashCommand` is used for fuzzy matching but is NOT shown in the autocomplete UI?

A) `name`
B) `description`
C) `hidden_keywords`
D) `aliases`

<details>
<summary>Answer</summary>

**C) `hidden_keywords`**

`hidden_keywords` is a space-separated string of terms used for fuzzy matching that don't appear in the autocomplete display. For example, `/clear` has `hidden_keywords="reset"` to match `/reset` without showing it.
</details>

---

## Question 3

What does `frozen=True` on the `SlashCommand` dataclass prevent?

A) Commands from being deleted
B) Commands from being added
C) Command attributes from being modified after creation
D) Commands from being used in multiple threads

<details>
<summary>Answer</summary>

**C) Command attributes from being modified after creation**

`frozen=True` makes the dataclass immutable, preventing accidental modification of command attributes after instantiation.
</details>

---

## Question 4

What is the purpose of the `aliases` field in `SlashCommand`?

A) Alternative spellings for fuzzy matching
B) Hidden keywords for fuzzy matching
C) Alternative command names that trigger the same handler
D) Backup command names if the primary fails

<details>
<summary>Answer</summary>

**C) Alternative command names that trigger the same handler**

Aliases like `("/q",)` for `/quit` are alternative names that trigger the same command handler. They are also added to the bypass frozensets automatically.
</details>

---

## Question 5

Which derived frozenset contains `/quit`?

A) `IMMEDIATE_UI`
B) `SIDE_EFFECT_FREE`
C) `QUEUE_BOUND`
D) `ALWAYS_IMMEDIATE`

<details>
<summary>Answer</summary>

**D) `ALWAYS_IMMEDIATE`**

`/quit` has `bypass_tier=BypassTier.ALWAYS`, so it's in `ALWAYS_IMMEDIATE` — commands that execute regardless of any busy state.
</details>

---

## Question 6

What scoring does a prefix match on the command name receive?

A) 90 points
B) 120 points
C) 150 points
D) 200 points

<details>
<summary>Answer</summary>

**D) 200 points**

From `SlashCommandController._score_command()`:
- Prefix match on command name: **200.0**
- Substring match on command name: **150.0**
- Hidden keyword match: **120.0**
- Description match: **90-110.0**
- Fuzzy ratio: **0-60.0**
</details>

---

## Question 7

Which file(s) must you update when adding a new slash command?

A) Just `command_registry.py`
B) Just `app.py`
C) `command_registry.py` and `app.py` only
D) `command_registry.py`, `app.py`, and `ui.py`

<details>
<summary>Answer</summary>

**D) `command_registry.py`, `app.py`, and `ui.py`**

Adding a slash command requires:
1. `command_registry.py` — Add `SlashCommand` to `COMMANDS`
2. `app.py` — Add handler branch in `_handle_command()`
3. `ui.py` — Update help screen in `show_help()`

A drift test verifies all three stay in sync.
</details>

---

## Question 8

What happens if you add a command to `COMMANDS` but forget to add its handler in `_handle_command()`?

A) The command silently does nothing
B) A runtime error occurs when the command is invoked
C) A drift test fails
D) The command is automatically added to the handler

<details>
<summary>Answer</summary>

**C) A drift test fails**

The `TestSlashCommandBypass.test_bypass_frozensets_match_handle_command` test extracts command literals from `_handle_command()` source and verifies each command in the frozensets has a handler. This catches missing handlers.
</details>

---

## Question 9

What bypass tier should be used for a command that opens a URL in the browser?

A) `ALWAYS`
B) `IMMEDIATE_UI`
C) `SIDE_EFFECT_FREE`
D) `QUEUED`

<details>
<summary>Answer</summary>

**C) `SIDE_EFFECT_FREE`**

`SIDE_EFFECT_FREE` is for commands whose side effect fires immediately (opening browser, URL) while chat output is deferred until idle. Examples: `/changelog`, `/docs`, `/feedback`.
</details>

---

## Question 10

What is the return type of `build_skill_commands()`?

A) `list[SlashCommand]`
B) `list[tuple[str, str]]`
C) `list[tuple[str, str, str]]`
D) `dict[str, SlashCommand]`

<details>
<summary>Answer</summary>

**C) `list[tuple[str, str, str]]`**

`build_skill_commands()` returns `list[tuple[str, str, str]]` — tuples of `(name, description, hidden_keywords)` matching the `SLASH_COMMANDS` format.
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **SlashCommand** — Frozen dataclass with name, description, bypass_tier, hidden_keywords, aliases
- **BypassTier** — Queue management: ALWAYS, CONNECTING, IMMEDIATE_UI, SIDE_EFFECT_FREE, QUEUED
- **CommandRegistry** — Single source of truth in `command_registry.py`
- **Fuzzy matching** — SequenceMatcher scoring with prefix > substring > keyword > description
- **Drift tests** — Verify sync between registry and handlers
- **Three-file update** — `command_registry.py`, `app.py`, `ui.py`

## Next Module

[Module 18: Polish for Production](../module-18-polish-production/README.md) — Final refinements, error handling, and release checklist.
