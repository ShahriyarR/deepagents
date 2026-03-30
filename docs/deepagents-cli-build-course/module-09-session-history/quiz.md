# Module 9 Quiz

Test your understanding of Session & History.

## Question 1

What is the primary purpose of a thread ID?

A) To encrypt message content
B) To uniquely identify a conversation session
C) To store messages in a database
D) To generate random numbers

<details>
<summary>Answer</summary>

**B) To uniquely identify a conversation session**

A thread ID ties all messages in a conversation together, allowing the system to retrieve and resume a specific conversation.

</details>

---

## Question 2

Which UUID version uses random generation?

A) UUID v1
B) UUID v4
C) UUID v7
D) None of the above

<details>
<summary>Answer</summary>

**B) UUID v4**

UUID v4 uses 122 bits of random data for generation. UUID v1 uses timestamp + MAC, and UUID v7 uses timestamp + random but with time ordering.

</details>

---

## Question 3

What is the key advantage of aiosqlite over sqlite3?

A) aiosqlite is faster
B) aiosqlite supports async/await operations
C) aiosqlite uses less memory
D) aiosqlite supports more SQL features

<details>
<summary>Answer</summary>

**B) aiosqlite supports async/await operations**

aiosqlite provides async versions of SQLite operations, allowing non-blocking database access in async applications.

</details>

---

## Question 4

What does "prepending history" mean?

A) Appending messages to the end of a conversation
B) Injecting prior messages before the current message
C) Deleting old messages
D) Encrypting message history

<details>
<summary>Answer</summary>

**B) Injecting prior messages before the current message**

Prepending history means adding historical messages to the prompt before the current message, giving the model context from the conversation.

</details>

---

## Question 5

In the database schema, what is the relationship between `threads` and `messages` tables?

A) One-to-one
B) One-to-many (one thread has many messages)
C) Many-to-many
D) No relationship

<details>
<summary>Answer</summary>

**B) One-to-many (one thread has many messages)**

A thread can have multiple messages, but each message belongs to exactly one thread. This is implemented with a foreign key: `messages.thread_id` references `threads.thread_id`.

</details>

---

## Question 6

What is the purpose of LangGraph checkpointing?

A) To speed up graph execution
B) To save and resume agent state at any point
C) To validate graph structure
D) To encrypt sensitive data

<details>
<summary>Answer</summary>

**B) To save and resume agent state at any point**

Checkpointing allows LangGraph to persist state after each step, enabling resumption of interrupted sessions and replay of historical states.

</details>

---

## Question 7

Which checkpointer type is best for a personal CLI tool?

A) MemorySaver
B) PostgresSaver
C) SqliteSaver
D) RedisSaver

<details>
<summary>Answer</summary>

**C) SqliteSaver**

For a personal CLI, SqliteSaver provides persistent storage without requiring a running database server, making it ideal for single-user applications.

</details>

---

## Question 8

What happens if you prepend too much history to a prompt?

A) Nothing, more history is always better
B) The model may exceed its context window
C) History gets automatically summarized
D) The database runs out of space

<details>
<summary>Answer</summary>

**B) The model may exceed its context window**

Models have limited context windows. Prepending too much history can exceed this limit and cause errors. Implement message or token limits.

</details>

---

## Question 9

How do you resume a conversation using thread IDs?

A) The thread ID is automatically stored in the model
B) Pass the thread ID to retrieve stored messages and prepend them
C) Thread IDs cannot be used to resume conversations
D) You must use the same process to resume

<details>
<summary>Answer</summary>

**B) Pass the thread ID to retrieve stored messages and prepend them**

Given a thread ID, you query the database for messages, prepend them to the current prompt, and continue the conversation with full context.

</details>

---

## Question 10

What is the difference between checkpointing and history storage?

A) They are the same thing
B) Checkpointing persists full agent state; history storage persists messages
C) History storage is faster than checkpointing
D) Checkpointing cannot be used with SQLite

<details>
<summary>Answer</summary>

**B) Checkpointing persists full agent state; history storage persists messages**

Checkpointing saves the entire LangGraph state (variables, messages, etc.) for replay/resume. History storage specifically stores message transcripts in a database.

</details>

---

## Question 11

What does `Session.touch()` do?

A) Deletes the session
B) Updates the `updated_at` timestamp
C) Creates a new session
D) Exports the session to JSON

<details>
<summary>Answer</summary>

**B) Updates the `updated_at` timestamp**

`touch()` is called to mark that a session was accessed or modified, updating its last-modified timestamp.

</details>

---

## Question 12

Why is it important to validate thread ID format?

A) It improves performance
B) It prevents invalid UUIDs from causing errors
C) It encrypts the thread ID
D) It is not important

<details>
<summary>Answer</summary>

**B) It prevents invalid UUIDs from causing errors**

Validating thread ID format ensures that only properly formatted UUIDs are accepted, preventing database errors and unexpected behavior.

</details>

---

## Summary

Review any questions you got wrong. Key concepts:

- **Thread ID** — Unique conversation identifier using UUID
- **SQLite + aiosqlite** — Async persistent storage for messages
- **Message history** — Stored transcript of conversations
- **Prepending** — Injecting history into prompts for context
- **Checkpointing** — LangGraph's built-in state persistence
- **Token limits** — Prevent context overflow from too much history

## Next Module

[Module 10: Skills System](../module-10-skills-system/README.md) — Load custom behaviors from SKILL.md files.
