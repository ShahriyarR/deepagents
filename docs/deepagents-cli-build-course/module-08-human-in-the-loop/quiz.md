# Module 8 Quiz

Test your understanding of Human-in-the-Loop concepts.

## Question 1

What does `interrupt()` do in LangGraph?

A) Terminates the graph entirely
B) Pauses execution and returns control to the caller
C) Logs a warning and continues
D) Switches to a different graph node

<details>
<summary>Answer</summary>

**B) Pauses execution and returns control to the caller**

`interrupt()` pauses the graph at that point and returns control to the Python caller. The caller can then resume the graph with a decision via `Command(resume=...)`.
</details>

---

## Question 2

Which `interrupt_on` value makes a tool always pause for approval?

A) `"never"`
B) `False`
C) `True`
D) `null`

<details>
<summary>Answer</summary>

**C) True**

Setting `interrupt_on: True` means the tool always triggers an interrupt for user approval. `False` means never interrupt, and an `InterruptOnConfig` object allows custom configuration.
</details>

---

## Question 3

In the bouncer metaphor, what does the bouncer represent?

A) The LLM model
B) The human user who approves dangerous actions
C) The shell command allowlist
D) The Textual UI framework

<details>
<summary>Answer</summary>

**B) The human user who approves dangerous actions**

The bouncer represents the human-in-the-loop — a person who reviews potentially dangerous operations before they execute, not blindly trusting the agent's request.
</details>

---

## Question 4

What is the correct way to resume a graph after an interrupt with an "approve" decision?

A) `graph.resume("approve")`
B) `graph.invoke(Command(resume="approve"))`
C) `interrupt(resume="approve")`
D) `graph.continue("approve")`

<details>
<summary>Answer</summary>

**B) `graph.invoke(Command(resume="approve"))`**

Use `Command(resume="approve")` passed to `graph.invoke()` or `graph.ape()` to resume execution after an interrupt.
</details>

---

## Question 5

Which tool would MOST likely be configured with `interrupt_on: False`?

A) `execute` (shell commands)
B) `write_file` (file writes)
C) `read_file` (file reads)
D) `edit_file` (file edits)

<details>
<summary>Answer</summary>

**C) `read_file` (file reads)**

Read-only operations are low-risk and typically don't require approval. `execute`, `write_file`, and `edit_file` all have side effects and thus require approval.
</details>

---

## Question 6

What does the `--auto-approve` flag do?

A) Enables the Textual UI
B) Skips all approval prompts and runs tools automatically
C) Deletes temporary files automatically
D) Forces the agent to use Claude Sonnet

<details>
<summary>Answer</summary>

**B) Skips all approval prompts and runs tools automatically**

When `--auto-approve` is enabled, `auto_approve=True` is passed to `create_cli_agent`, which sets `interrupt_on: {}` (empty), bypassing all interrupts. This is intended for CI/CD and batch scripts.
</details>

---

## Question 7

How can a user toggle auto-approve mode at runtime in the Textual UI?

A) `ctrl+a`
B) `ctrl+shift+a`
C) `shift+tab`
D) `escape`

<details>
<summary>Answer</summary>

**C) `shift+tab`**

The binding `"shift+tab"` triggers `action_toggle_auto_approve()`, which flips the auto-approve state and updates the status bar.
</details>

---

## Question 8

What does `InterruptOnConfig` contain?

A) The tool's implementation code
B) `allowed_decisions` and `description`
C) The user's API key
D) The LLM model name

<details>
<summary>Answer</summary>

**B) `allowed_decisions` and `description`**

`InterruptOnConfig` is a TypedDict with `allowed_decisions` (list of valid decisions like `["approve", "reject"]`) and `description` (what the tool does).
</details>

---

## Question 9

Why is `--auto-approve` not the default setting?

A) It's too slow to check
B) It would defeat the security purpose of human-in-the-loop
C) It's only available in enterprise editions
D) The flag doesn't exist

<details>
<summary>Answer</summary>

**B) It would defeat the security purpose of human-in-the-loop**

Auto-approve bypasses all safety checks. It should be an explicit opt-in for CI/batch scenarios, not a default, because the whole point of HITL is to have a human verify dangerous operations.
</details>

---

## Question 10

What is the purpose of `_add_interrupt_on()`?

A) To add a new tool to the agent
B) To configure which tools require approval for which operations
C) To install Python packages
D) To format error messages

<details>
<summary>Answer</summary>

**B) To configure which tools require approval for which operations**

`_add_interrupt_on()` is a function that returns a dictionary mapping tool names to their `InterruptOnConfig`, defining which tools should trigger interrupts and what information to show in approval prompts.
</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **`interrupt()`** pauses LangGraph for external input
- **`interrupt_on`** configures per-tool approval requirements
- **`InterruptOnConfig`** defines allowed decisions and descriptions
- **Human-in-the-loop** adds targeted friction for dangerous operations
- **`--auto-approve`** bypasses interrupts for CI/batch use
- **Shift+Tab** toggles auto-approve at runtime

## Next Module

[Module 9: Session & History](../module-09-session-history/README.md) — Thread IDs and SQLite persistence for conversation history.
