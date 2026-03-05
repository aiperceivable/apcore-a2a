# apcore-a2a

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python Version](https://img.shields.io/badge/python-3.10%2B-blue)](https://www.python.org/downloads/)

**apcore-a2a** is an automatic [A2A (Agent-to-Agent)](https://google.github.io/A2A/) protocol adapter for the [apcore](https://github.com/aipartnerup/apcore-python) ecosystem. It allows you to expose any apcore Module Registry as a fully functional, standards-compliant A2A agent with zero manual effort.

By reading the existing apcore metadata—including `input_schema`, `output_schema`, descriptions, and behavioral annotations—apcore-a2a eliminates the need to hand-write Agent Cards, JSON-RPC endpoints, and task lifecycle logic.

---

## Key Features

- **🚀 One-Call Server**: Launch a compliant A2A server using `serve(registry)`.
- **📇 Auto Agent Card**: Generates `/.well-known/agent.json` automatically from your module metadata.
- **🛠️ Skill Mapping**: Converts apcore modules into A2A Skills, preserving schemas and examples.
- **🔄 Task Lifecycle**: Full implementation of the A2A task state machine (submitted, working, completed, failed, etc.).
- **📡 Streaming & SSE**: Native support for `message/stream` with real-time status and artifact updates.
- **🔗 A2A Client**: A built-in `A2AClient` to discover and invoke other A2A agents.
- **🛡️ Enterprise Ready**: JWT/Bearer authentication bridged directly to apcore's ACL (Identity) system.
- **🔔 Push Notifications**: Support for webhook-based task state updates.

---

## Installation

```bash
pip install apcore-a2a
```

*Note: Requires Python 3.10+ and apcore-python 0.6.0+.*

---

## Quick Start

### 1. Serve your modules as an A2A Agent

If you already have apcore modules, you can expose them as an A2A agent in just a few lines:

```python
from apcore import Registry
from apcore_a2a import serve

# 1. Initialize your registry and discover modules
registry = Registry(extensions_dir="./extensions")
registry.discover()

# 2. Start the A2A server
serve(registry, name="My Assistant Agent", host="0.0.0.0", port=8000)
```

Your agent is now discoverable at `http://localhost:8000/.well-known/agent.json`.

### 2. Call a remote A2A Agent

Use the `A2AClient` to interact with any A2A-compliant agent:

```python
import asyncio
from apcore_a2a import A2AClient

async def main():
    async with A2AClient("http://remote-agent:8000") as client:
        # Discover skills
        card = await client.agent_card
        print(f"Connected to {card['name']} with {len(card['skills'])} skills")

        # Send a message
        task = await client.send_message({
            "role": "user",
            "parts": [{"type": "text", "text": "Hello, how can you help?"}]
        }, metadata={"skillId": "general.chat"})
        
        print(f"Task status: {task['status']['state']}")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Architecture

apcore-a2a acts as a thin, protocol-specific layer on top of `apcore-python`. It maps A2A concepts to apcore primitives:

| A2A Concept | apcore Mapping |
|-------------|----------------|
| **Agent Card** | Derived from Registry configuration |
| **Skill** | Maps 1:1 to an apcore Module |
| **Task** | Managed execution of `Executor.call_async()` |
| **Streaming** | Wrapped `Executor.stream()` via SSE |
| **Security** | Bridged to apcore's `Identity` context |

---

## Documentation

Detailed documentation can be found in the `docs/` directory:

- [Product Requirements (PRD)](docs/apcore-a2a/prd.md)
- [Technical Design (TDD)](docs/apcore-a2a/tech-design.md)
- [Software Requirements (SRS)](docs/apcore-a2a/srs.md)

---

## License

This project is licensed under the **Apache License 2.0**. See the [LICENSE](LICENSE) file for details.
