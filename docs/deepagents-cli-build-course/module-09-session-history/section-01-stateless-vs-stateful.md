# Section 1: Stateless vs Stateful

Understand the difference between stateless and stateful agent architectures.

## Stateless Architecture

In a **stateless** system, each request is independent:

```
Request 1: "My name is Alice"
Response 1: "Hi Alice!"

Request 2: "What's my name?"
Response 2: "I don't know, you didn't tell me."
```

Every message contains all context needed to respond. The system has no memory of previous interactions.

### Problems with Stateless

- **No continuity** — Every conversation starts from scratch
- **Repetitive context** — Users must re-explain things
- **Broken workflows** — Multi-step tasks fail mid-way
- **Lost context** — Important details forgotten

## Stateful Architecture

In a **stateful** system, state persists across requests:

```
Request 1: "My name is Alice"
Response 1: "Hi Alice!"
(State stored: user_name = "Alice")

Request 2: "What's my name?"
Response 2: "Your name is Alice."
(State retrieved: user_name = "Alice")
```

The system maintains context that survives across messages.

### Types of State

| Type | Description | Example |
|------|-------------|---------|
| **Session state** | Per-conversation data | Thread ID, user name |
| **Message history** | Conversation transcript | All messages in thread |
| **Agent memory** | Long-term learned facts | User preferences |
| **Tool context** | Active file edits, shell state | Current working directory |

## Architecture Comparison

### Stateless Flow

```
┌─────────────────────────────────────────────────────────────┐
│  User: "My name is Alice"                                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  LLM (no context)                                           │
│  System: "You are a helpful assistant."                      │
│  Human: "My name is Alice"                                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Response: "Hi Alice!"                                      │
└─────────────────────────────────────────────────────────────┘
                              │
                         (Session ends)
                              │
                              ▼
                    ┌─────────────────┐
                    │   No memory     │
                    └─────────────────┘
```

### Stateful Flow

```
┌─────────────────────────────────────────────────────────────┐
│  User: "My name is Alice"                                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Session Store                                              │
│  ┌─────────────────────────────────────────┐               │
│  │ thread_abc123:                          │               │
│  │   messages: [                           │               │
│  │     {"role": "user", "content": "..."} │               │
│  │   ]                                    │               │
│  │   user_name: "Alice"                    │               │
│  └─────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  LLM (with history)                                         │
│  System: "You are a helpful assistant."                     │
│  Human: "My name is Alice"                                  │
│  [History prepended from session store]                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Response: "Hi Alice!"                                      │
│  (Session updated with new message)                         │
└─────────────────────────────────────────────────────────────┘
```

## Implementing State in Your CLI

### State Management Options

| Option | Pros | Cons | Best For |
|--------|------|------|----------|
| **In-memory** | Fast, simple | Lost on restart | Ephemeral testing |
| **SQLite** | Persistent, async, lightweight | Single-user | Personal CLI tools |
| **PostgreSQL** | Scalable, concurrent | Setup complexity | Production services |
| **Redis** | Very fast, ephemeral persistence | No durability | Caching layer |

For a personal CLI, SQLite is the sweet spot — persistent but simple.

## Session vs Thread

These terms are often used interchangeably, but there are subtle differences:

| Term | Focus | Lifespan |
|------|-------|----------|
| **Session** | User's connection | From connect to disconnect |
| **Thread** | A conversation topic | Can span multiple sessions |

In this module, we use **thread ID** to identify a conversation, which may span multiple sessions.

## Key Takeaways

- **Stateless** — Each request is independent, no memory
- **Stateful** — State persists across requests
- **Thread ID** — Unique identifier for a conversation
- **Message history** — Stored transcript of the conversation
- **SQLite** — Lightweight persistent storage for CLI use cases

## Next Section

[Generate Thread ID](./section-02-generate-thread-id.md) — Create unique session identifiers with UUID.
