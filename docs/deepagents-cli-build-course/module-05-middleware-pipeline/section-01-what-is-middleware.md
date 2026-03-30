# Section 1: What is Middleware?

Understanding the interceptor pattern for modifying agent behavior.

## The Problem

Sometimes you need to modify how an agent behaves **without changing the agent itself**. For example:

- Inject project-specific context into every LLM call
- Switch the model at runtime based on user preference
- Add skill documentation to the system prompt
- Log all LLM requests/responses for debugging

Changing the agent code for each of these would be messy. You need a **pluggable way** to intercept and modify behavior.

## The Solution: Middleware

Middleware is a **layer between your code and the LLM** that can intercept, modify, and act on every request/response:

```
Your Code → Middleware → Middleware → ... → LLM
          ← Middleware ← Middleware ← ... ← LLM
```

Each middleware can:
1. **See the request** before it goes to the LLM
2. **Modify the request** (add context, change model, etc.)
3. **Pass it along** to the next middleware or LLM
4. **See the response** and potentially modify it

## Real-World Analogy: HTTP Middleware

If you've used web frameworks like Express.js or Django, you know middleware:

```
Request → Logger Middleware → Auth Middleware → Route Handler
        ← Logger Middleware ← Auth Middleware ← Route Handler
```

Each middleware in the chain can:
- Log the request
- Check authentication
- Modify the request
- Modify the response

Agent middleware works the same way!

## Why Middleware for Agents?

| Use Case | Without Middleware | With Middleware |
|----------|-------------------|-----------------|
| Add context | Modify agent code | Add `ContextMiddleware` |
| Switch models | Recompile graph | Add `ModelMiddleware` |
| Load skills | Hardcode in prompt | Add `SkillsMiddleware` |
| Debug calls | Add logging everywhere | Add `LoggingMiddleware` |

## LangChain Agent Middleware

LangChain Agents provides the `AgentMiddleware` base class:

```python
from langchain.agents.middleware.types import AgentMiddleware

class MyMiddleware(AgentMiddleware):
    def wrap_model_call(self, request, handler):
        # 1. See/modify request
        modified_request = self.modify_request(request)
        
        # 2. Call next middleware/llm
        response = handler(modified_request)
        
        # 3. Optionally modify response
        return response
```

## The Request Flow

```
1. Your code calls: agent.invoke({"messages": [...]})

2. LangGraph processes the state and decides to call the LLM

3. The middleware pipeline intercepts:
   
   ┌─────────────────────────────────────────────┐
   │  ConfigurableModelMiddleware.wrap_model_call │
   │  - Applies runtime model overrides          │
   │  - Calls handler (next middleware/llm)       │
   └─────────────────────┬───────────────────────┘
                         │
                         ▼
   ┌─────────────────────────────────────────────┐
   │  MemoryMiddleware.wrap_model_call            │
   │  - Loads AGENTS.md files                     │
   │  - Injects context into system prompt        │
   │  - Calls handler (next middleware/llm)       │
   └─────────────────────┬───────────────────────┘
                         │
                         ▼
   ┌─────────────────────────────────────────────┐
   │  SkillsMiddleware.wrap_model_call            │
   │  - Loads skill documentation                 │
   │  - Injects skills into system prompt        │
   │  - Calls handler (actual LLM)               │
   └─────────────────────┬───────────────────────┘
                         │
                         ▼
   ┌─────────────────────────────────────────────┐
   │  The LLM processes the (possibly modified)   │
   │  request and returns a response              │
   └─────────────────────┬───────────────────────┘
                         │
   (Responses bubble back up through the chain)
```

## Middleware Ordering

**Order matters!** Middleware is called in registration order for modification, but the "inner" (actual LLM call) is reached last:

```
middleware = [A, B, C]

# Request flow: A → B → C → LLM
# Response flow: LLM → C → B → A
```

The last middleware registered wraps the LLM most closely.

## What Can Middleware Modify?

1. **System prompt** — Add context, instructions, or documentation
2. **Model selection** — Change which model handles the request
3. **Model settings** — Adjust temperature, max tokens, etc.
4. **Request metadata** — Add custom fields for logging/analytics
5. **Response** — Modify what the agent sees

## When to Use Middleware

Use middleware when you need to:

- **Add behavior to every agent call** (context injection, logging)
- **Modify requests conditionally** (runtime model switching)
- **Separate concerns** (don't mix logging in your agent logic)
- **Reuse across agents** (middleware can be shared)

Use direct agent modification when:

- The behavior is specific to one agent
- It's core to how the agent works
- It needs access to internal agent state

## Key Takeaways

- **Middleware intercepts** requests before they reach the LLM
- **Chain pattern** — Each middleware calls the next until the LLM
- **Response bubbles back** — Can modify responses too
- **Order matters** — Last registered wraps the LLM most closely
- **Separation of concerns** — Keep middleware focused and reusable

## Next Section

[Build the Base Class](./section-02-build-base-class.md) — Create your first AgentMiddleware subclass.
