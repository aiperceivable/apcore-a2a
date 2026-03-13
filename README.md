<div align="center">
  <img src="./apcore-a2a-logo.svg" alt="apcore logo" width="200"/>
</div>

# apcore-a2a

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python Version](https://img.shields.io/badge/python-3.11%2B-blue)](https://www.python.org/downloads/)
[![TypeScript](https://img.shields.io/badge/TypeScript-Node_18%2B-blue)](https://github.com/aipartnerup/apcore-a2a-typescript)

> **Build once, invoke by Code or AI.**

**apcore-a2a** is an automatic [A2A (Agent-to-Agent)](https://google.github.io/A2A/) protocol adapter for the [apcore](https://github.com/aipartnerup/apcore) ecosystem. It allows you to expose any apcore Module Registry as a fully functional, standards-compliant A2A agent with zero manual effort.

### 🚀 Official SDKs

- **Python SDK**: [aipartnerup/apcore-a2a-python](https://github.com/aipartnerup/apcore-a2a-python)
- **TypeScript SDK**: [aipartnerup/apcore-a2a-typescript](https://github.com/aipartnerup/apcore-a2a-typescript)

---

By reading the existing apcore metadata—including `input_schema`, `output_schema`, descriptions, behavioral annotations (readonly, cacheable, paginated, etc.), and AI metadata conventions (x-preconditions, x-cost-per-call, etc.)—apcore-a2a eliminates the need to hand-write Agent Cards, JSON-RPC endpoints, and task lifecycle logic.

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

=== "Python"
    ```bash
    pip install apcore-a2a
    ```
    *Requires Python 3.11+ and apcore 0.9.0+.*

=== "TypeScript"
    ```bash
    npm install apcore-a2a
    ```
    *Requires Node.js 18+ and apcore-js 0.8.0+.*

---

## Quick Start

### 1. Serve your modules as an A2A Agent

If you already have apcore modules, you can expose them as an A2A agent in just a few lines:

=== "Python"
    ```python
    from apcore import Registry
    from apcore_a2a import serve

    # 1. Initialize your registry and discover modules
    registry = Registry(extensions_dir="./extensions")
    registry.discover()

    # 2. Start the A2A server
    serve(registry, name="My Assistant Agent", host="0.0.0.0", port=8000)
    ```

=== "TypeScript"
    ```typescript
    import { Registry } from "apcore-js";
    import { serve } from "apcore-a2a";

    const registry = new Registry({ extensionsDir: "./extensions" });
    await registry.discover();

    serve(registry, {
      name: "My Assistant Agent",
      port: 8000,
    });
    ```

### 2. Call a remote A2A Agent

Use the `A2AClient` to interact with any A2A-compliant agent:

=== "Python"
    ```python
    import asyncio
    from apcore_a2a import A2AClient

    async def main():
        async with A2AClient("http://remote-agent:8000") as client:
            # Discover skills
            card = await client.discover()
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

=== "TypeScript"
    ```typescript
    import { A2AClient } from "apcore-a2a";

    const client = new A2AClient("http://remote-agent:8000");

    // Discover skills
    const card = await client.discover();
    console.log(`Connected to ${card.name}`);

    // Send a message
    const task = await client.sendMessage(
      { role: "user", parts: [{ type: "text", text: "Hello, how can you help?" }] },
      { metadata: { skillId: "general.chat" } }
    );
    
    console.log(`Task status: ${task.status.state}`);
    ```

---

## Architecture

apcore-a2a acts as a thin, protocol-specific layer on top of `apcore-python`. It maps A2A concepts to apcore primitives:

| A2A Concept | apcore Mapping |
|-------------|----------------|
| **Agent Card** | Derived from Registry configuration |
| **Skill** | Maps 1:1 to an apcore Module |
| **Skill.tags** | `module.tags` |
| **Skill metadata** | `annotations` (cacheable, paginated) + `metadata` (x-preconditions, x-cost-per-call, x-output-source) |
| **Task** | Managed execution of `Executor.call_async()` |
| **Streaming** | Wrapped `Executor.stream()` via SSE |
| **Security** | Bridged to apcore's `Identity` context |

---

## Documentation

- **[Full Documentation Site](https://aipartnerup.github.io/apcore-a2a/)**
- **[Getting Started Guide](docs/getting-started.md)** — Install, serve, and call agents
- [Feature Specs Overview](docs/features/overview.md) — All feature specifications
- [Specifications (PRD, TDD, SRS)](docs/spec/)

---

## License

This project is licensed under the **Apache License 2.0**. See the [LICENSE](LICENSE) file for details.
