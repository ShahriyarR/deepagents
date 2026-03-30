# Section 1: What is Streaming?

Understanding the difference between streaming and blocking LLM responses.

## Two Ways to Get LLM Responses

When you call an LLM, you have two options:

### 1. Invoke (Blocking)

```python
response = model.invoke("What is 2+2?")
print(response.content)  # "2"
```

The entire response is returned **after it's complete**. This takes several seconds for longer responses.

### 2. Stream (Real-time)

```python
for token in model.stream("What is 2+2?"):
    print(token.content, end="", flush=True)
# Shows: 2 (as each character arrives)
```

Tokens are returned **as they're generated**. You see the response appear character by character.

## Why Does Streaming Matter?

| Aspect | Invoke | Stream |
|--------|--------|--------|
| **Latency perception** | High (wait for full response) | Low (see progress immediately) |
| **User experience** | Feels slow for long responses | Feels responsive |
| **Use case** | Batch processing, short prompts | Interactive chat, long outputs |
| **Code complexity** | Simple | Slightly more complex |

## For an Interactive CLI

Streaming is **essential**. Users expect to see:

```
>>> Write a Python function
Here's a Python function that does X:

def example():
    pass
```

With invoke, users wait 5-10 seconds then see everything. With streaming, they see the response appear in real-time.

## How LLM Streaming Works

LLMs generate tokens **one at a time**:

```
"What" → "is" → "2" → "+" → "2" → "?" → "=" → " " → "4"
```

Each `→` is a token. Streaming sends each token as it's generated.

## Token Sizes

- English: ~4 characters per token (average)
- Code: ~2-3 characters per token (more tokens!)
- Chinese/Japanese: 1 token per character

A 100-word response might be 150-250 tokens.

## Async vs Sync Streaming

LangChain supports both:

```python
# Sync streaming
for token in model.stream("hello"):
    print(token.content)

# Async streaming
async for token in model.astream("hello"):
    print(token.content)
```

For a CLI, **async** is preferred because you can handle other events while waiting.

## Key Takeaways

- **Invoke** waits for the complete response before returning
- **Stream** yields tokens as they're generated
- Streaming gives **better perceived performance** for interactive use
- For CLI apps, streaming is essential for good UX

## Next Section

[Build the REPL Skeleton](./section-02-build-repl-skeleton.md) — Create the input loop.
