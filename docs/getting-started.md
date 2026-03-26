# Getting Started

This guide walks you through installing **apcore-a2a**, serving your first A2A agent, and calling remote agents — in both Python and TypeScript.

## Prerequisites

You need an existing apcore project with at least one module defined:
- **Python**: [apcore-python](https://github.com/aiperceivable/apcore-python) — see the [Getting Started guide](https://aiperceivable.github.io/apcore/getting-started.html)
- **TypeScript**: [apcore-js](https://github.com/aiperceivable/apcore-typescript)

---

## Installation

=== "Python"

    ```bash
    pip install apcore-a2a
    ```

    Requires Python 3.11+ and `apcore` 0.9.0+.

=== "TypeScript"

    ```bash
    npm install apcore-a2a
    # or
    pnpm add apcore-a2a
    ```

    Requires Node.js 18+ and `apcore-js` 0.8.0+.

---

## Step 1: Serve Your Modules as an A2A Agent

=== "Python"

    ```python
    from apcore import Registry
    from apcore_a2a import serve

    # 1. Set up your registry
    registry = Registry(extensions_dir="./extensions")
    registry.discover()

    # 2. Launch the A2A server (blocks until shutdown)
    serve(registry, name="My Agent", port=8000)
    ```

=== "TypeScript"

    ```typescript
    import { Registry } from "apcore-js";
    import { serve } from "apcore-a2a";

    const registry = new Registry({ extensionsDir: "./extensions" });
    await registry.discover();

    serve(registry, {
      name: "My Agent",
      port: 8000,
    });
    ```

Your agent is now running. Verify by visiting:

=== "Python"

    ```
    http://localhost:8000/.well-known/agent.json
    ```

=== "TypeScript"

    ```
    http://localhost:8000/.well-known/agent-card.json
    ```

This returns the auto-generated **Agent Card** — a JSON document describing your agent's name, skills, and capabilities, all derived from your apcore module metadata.

---

## Step 2: Define a Module

If you don't have existing modules, here's the fastest way to create one:

=== "Python"

    ```python
    import apcore
    from apcore_a2a import serve

    @apcore.module(id="greeting.hello", description="Say hello to someone")
    def hello(name: str) -> str:
        return f"Hello, {name}! Welcome to the A2A world."

    # apcore auto-registers the module in the global registry
    serve(apcore.registry, name="Greeter Agent", port=8000)
    ```

=== "TypeScript"

    ```typescript
    import { Type } from "@sinclair/typebox";
    import { FunctionModule, Registry } from "apcore-js";
    import { serve } from "apcore-a2a";

    const hello = new FunctionModule({
      moduleId: "greeting.hello",
      description: "Say hello to someone",
      inputSchema: Type.Object({ name: Type.String() }),
      outputSchema: Type.Object({ result: Type.String() }),
      execute: async (inputs) => ({ result: `Hello, ${inputs.name}! Welcome to the A2A world.` }),
    });

    const registry = new Registry();
    registry.register(hello);

    serve(registry, { name: "Greeter Agent", port: 8000 });
    ```

---

## Step 3: Call a Remote A2A Agent

Use `A2AClient` to discover and invoke any A2A-compliant agent.

=== "Python"

    ```python
    import asyncio
    from apcore_a2a import A2AClient

    async def main():
        async with A2AClient("http://localhost:8000") as client:
            # Discover the agent
            card = await client.discover()
            print(f"Agent: {card['name']}, Skills: {len(card['skills'])}")

            # Send a message
            task = await client.send_message(
                {"role": "user", "parts": [{"type": "text", "text": "Hello from Python!"}]},
                metadata={"skillId": "greeting.hello"},
            )
            print(f"Result: {task['status']['state']}")

    asyncio.run(main())
    ```

=== "TypeScript"

    ```typescript
    import { A2AClient } from "apcore-a2a";

    const client = new A2AClient("http://localhost:8000");

    // Discover the agent
    const card = await client.discover();
    const skills = card["skills"] as unknown[];
    console.log(`Agent: ${card["name"]}, Skills: ${skills.length}`);

    // Send a message
    const task = await client.sendMessage(
      { role: "user", parts: [{ type: "text", text: "Hello from TypeScript!" }] },
      { metadata: { skillId: "greeting.hello" } },
    );
    const status = task["status"] as Record<string, unknown>;
    console.log(`Result: ${status["state"]}`);

    client.close();
    ```

---

## Step 4: Streaming Responses

For long-running tasks, use SSE streaming to receive real-time updates.

=== "Python"

    ```python
    from apcore_a2a import A2AClient

    async with A2AClient("http://localhost:8000") as client:
        async for event in client.stream_message(
            {"role": "user", "parts": [{"type": "text", "text": "Process this data"}]},
            metadata={"skillId": "data.process"},
        ):
            if "status" in event:
                print(f"Status: {event['status']['state']}")
            if "artifact" in event:
                print(f"Artifact: {event['artifact']}")
    ```

=== "TypeScript"

    ```typescript
    import { A2AClient } from "apcore-a2a";

    const client = new A2AClient("http://localhost:8000");

    for await (const event of client.streamMessage(
      { role: "user", parts: [{ type: "text", text: "Process this data" }] },
      { metadata: { skillId: "data.process" } },
    )) {
      if (event.status) {
        console.log(`Status: ${event.status.state}`);
      }
      if (event.artifact) {
        console.log(`Artifact:`, event.artifact);
      }
    }
    ```

---

## Step 5: Enable Authentication

Protect your agent with JWT/Bearer authentication.

=== "Python"

    ```python
    import os
    from apcore import Registry
    from apcore_a2a import serve
    from apcore_a2a.auth import JWTAuthenticator

    registry = Registry(extensions_dir="./extensions")
    registry.discover()

    auth = JWTAuthenticator(key=os.environ["APCORE_JWT_SECRET"])

    serve(
        registry,
        name="Secure Agent",
        auth=auth,
        port=8000,
    )
    ```

    Clients include the token in the `A2AClient` constructor:

    ```python
    async with A2AClient("http://localhost:8000", auth="Bearer eyJ...") as client:
        task = await client.send_message(...)
    ```

=== "TypeScript"

    ```typescript
    import { serve, JWTAuthenticator } from "apcore-a2a";

    const auth = new JWTAuthenticator({ key: process.env.APCORE_JWT_SECRET! });

    serve(registry, {
      name: "Secure Agent",
      auth,
      port: 8000,
    });
    ```

    Client side:

    ```typescript
    import { A2AClient } from "apcore-a2a";

    const client = new A2AClient("http://localhost:8000", { auth: "Bearer eyJ..." });
    ```

---

## Step 6: Use the CLI (No Code Required)

Launch an A2A agent directly from the command line.

=== "Python"

    ```bash
    apcore-a2a serve --extensions-dir ./extensions --name "CLI Agent" --port 8000
    ```

    With options:

    ```bash
    apcore-a2a serve \
      --extensions-dir ./extensions \
      --name "Production Agent" \
      --port 8080 \
      --explorer \
      --push-notifications \
      --log-level info
    ```

=== "TypeScript"

    ```bash
    npx apcore-a2a serve --extensions-dir ./extensions --name "CLI Agent" --port 8000
    ```

---

## Step 7: Embed in an Existing Web App

Get the underlying app object and mount it into your existing web framework.

=== "Python (FastAPI)"

    ```python
    from fastapi import FastAPI
    from apcore import Registry
    from apcore_a2a import async_serve

    app = FastAPI()
    registry = Registry(extensions_dir="./extensions")
    registry.discover()

    @app.on_event("startup")
    async def mount_a2a():
        a2a_app = await async_serve(registry, url="https://example.com/a2a")
        app.mount("/a2a", a2a_app)
    ```

=== "TypeScript (Express)"

    ```typescript
    import express from "express";
    import { asyncServe } from "apcore-a2a";

    const outer = express();

    const a2aApp = await asyncServe(registry, { url: "https://example.com/a2a" });
    outer.use("/a2a", a2aApp);

    outer.listen(8000);
    ```

---

## Configuration Reference

### `serve()` Options

| Option | Python | TypeScript | Default | Description |
|--------|--------|------------|---------|-------------|
| Host | `host` | `host` | `"0.0.0.0"` (CLI: `"127.0.0.1"`) | Bind address |
| Port | `port` | `port` | `8000` | Bind port |
| Name | `name` | `name` | From registry or `"Apcore Agent"` | Agent display name |
| Description | `description` | `description` | From registry | Agent description |
| Version | `version` | `version` | From registry | Semver version |
| URL | `url` | `url` | `http://{host}:{port}` | Public base URL |
| Auth | `auth` | `auth` | `None` | Authenticator instance |
| Storage | `task_store` | `taskStore` | In-memory | Task persistence backend |
| CORS | `cors_origins` | `corsOrigins` | `None` | Allowed CORS origins |
| Push | `push_notifications` | — | `false` | Enable webhook notifications (Python only) |
| Explorer | `explorer` | `explorer` | `false` | Enable Explorer UI |
| Metrics | `metrics` | `metrics` | `false` | Enable `/metrics` endpoint |
| Log Level | `log_level` | `logLevel` | `"info"` | Logging level |

---

## What's Next?

- **Explorer UI**: Enable `explorer=True` to get an interactive browser UI at `/explorer` for testing your agent
- **Push Notifications**: Enable `push_notifications=True` for webhook-based async task updates
- **Multi-Agent Workflows**: Use `A2AClient` to orchestrate multiple agents in complex pipelines
- **Custom Storage**: Implement the `TaskStore` protocol for Redis/PostgreSQL persistence
- **Detailed Design**: See the [Technical Design](spec/tech-design.md) for an architecture deep-dive.
- **Feature Specs**: See the [Feature Specs Overview](features/overview.md) for implementation details.
