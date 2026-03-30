# Section 1: Local vs Remote Execution

When and why to run code in cloud environments instead of locally.

## The Local Execution Model

So far, the CLI executes code locally using `asyncio.create_subprocess_exec`. This works well for:

- Quick prototyping and testing
- Small scripts that don't require special resources
- Environments where internet access is limited
- Cost-sensitive workloads

However, local execution has hard limits:

| Constraint | Impact |
|------------|--------|
| CPU cores | Limited by your machine |
| Memory | Limited by your machine |
| GPU | Not available on most machines |
| Persistence | Environment resets between runs |
| Isolation | Poor - code runs in your environment |
| Cold starts | None for local |

## When Remote Execution Makes Sense

Remote sandboxes excel when you need:

**1. GPU Access**
```python
# Local: No GPU
# Remote: Full GPU access (A100, H100)
code = """
import torch
print(f"GPU: {torch.cuda.get_device_name(0)}")
"""
```

**2. Persistent Environments**
```python
# Daytona/Longtail: State persists between executions
# Install once, use forever
code = """
!pip install rare-model
from rare_model import Inferencer
inference = Inferencer()  # Already installed
"""
```

**3. Heavy Compute**
```bash
# Training a model locally might take hours
# Remote sandbox parallelizes instantly
$ deepagents run --sandbox modal --model training.py
```

**4. Security Isolation**
```python
# Untrusted code runs in isolated cloud VM
# No access to your local filesystem or credentials
result = await sandbox.execute(untrusted_code, language="python")
```

**5. Cross-language Support**
```python
# Some sandboxes support multiple languages
result = await sandbox.execute("#!/bin/bash\necho hello", language="bash")
result = await sandbox.execute("SELECT * FROM users", language="sql")
```

## Remote Sandbox Providers

| Provider | Strengths | Best For |
|----------|-----------|----------|
| **LangSmith** | Integration with LangChain, streaming | LangChain applications |
| **Daytona** | Persistent environments, pre-installed tools | Full dev environments |
| **Modal** | GPU support, serverless scale, fast cold starts | ML inference, training |
| **E2B** | Security-first, browser tooling | Untrusted code execution |
| **CodeInterpreter** | Structured outputs, file handling | Data analysis |

## Architecture Decision: Why a Protocol?

We use the **Protocol pattern** (from Module 12) to decouple the CLI from specific sandbox providers:

```
CLI Code
    │
    ▼
SandboxBackendProtocol  ◄─── abstract interface
    │
    ├──▶ LocalBackend     (current subprocess-based)
    ├──▶ LangSmithBackend (cloud VM)
    ├──▶ DaytonaBackend   (managed dev)
    └──▶ ModalBackend     (serverless)

Each backend implements:
- connect() -> None
- execute(code, language) -> ExecutionResult
- disconnect() -> None
```

This means:
- Users choose their backend via `--sandbox` flag
- Adding new backends requires no changes to CLI core
- We can default to local for simplicity, cloud for power

## Comparing Execution Characteristics

| Aspect | Local | LangSmith | Daytona | Modal |
|--------|-------|-----------|---------|-------|
| Cold start | Instant | 2-5s | 1-3s | 0.5-2s |
| GPU | No | Yes | Optional | Yes |
| Persistence | None | Session | Full env | Ephemeral |
| Cost | Free | Pay-per-use | Subscription | Pay-per-use |
| Streaming | N/A | Yes | Yes | Limited |

## The `--sandbox` Flag

We add a simple CLI flag to select execution backend:

```python
# main.py
run_parser.add_argument(
    "--sandbox",
    choices=["local", "langsmith", "daytona", "modal"],
    default="local",
    help="Execution backend (default: local)",
)
```

```bash
# Default: local execution
$ deepagents run

# Cloud: LangSmith sandbox
$ deepagents run --sandbox langsmith

# Managed dev environment
$ deepagents run --sandbox daytona

# Serverless compute
$ deepagents run --sandbox modal
```

## Security Considerations

Remote sandboxes introduce new security concerns:

1. **Data egress** — Code runs on third-party infrastructure
2. **Credential exposure** — API keys may be needed in sandbox
3. **Network access** — Sandbox can make outbound connections
4. **Resource limits** — Need to prevent infinite loops, memory bombs

Each backend should implement:
- Execution timeouts
- Memory limits
- Network egress restrictions (where possible)
- No persistent credential storage

## Next Section

[Sandbox Interface](./section-02-sandbox-interface.md) — Design the `SandboxBackendProtocol` for swappable backends.
