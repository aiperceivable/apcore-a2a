---
description: "Getting started guide: install apcore-a2a, serve your first A2A agent, and call remote agents, with parallel Python, TypeScript, and Rust code examples and prerequisites."
---

# Getting Started

This guide walks you through installing **apcore-a2a**, serving your first A2A agent, and calling remote agents — in Python, TypeScript, and Rust.

## Prerequisites

You need an existing apcore project with at least one module defined:
- **Python**: [apcore-python](https://github.com/aiperceivable/apcore-python) — see the [Getting Started guide](https://aiperceivable.github.io/apcore/getting-started.html)
- **TypeScript**: [apcore-js](https://github.com/aiperceivable/apcore-typescript)
- **Rust**: [apcore-rust](https://github.com/aiperceivable/apcore-rust)

---

## Installation

=== "Python"

    ```bash
    pip install apcore-a2a
    ```

    Requires Python 3.11+ and `apcore` 0.22.0+.

=== "TypeScript"

    ```bash
    npm install apcore-a2a
    # or
    pnpm add apcore-a2a
    ```

    Requires Node.js 18+ and `apcore-js` 0.22.0+.

=== "Rust"

    ```bash
    cargo add apcore-a2a
    ```

    Or add it to `Cargo.toml`:

    ```toml
    [dependencies]
    apcore-a2a = "0.4"
    tokio = { version = "1", features = ["full"] }
    ```

    Requires Rust 1.75+ (edition 2021) and `apcore` 0.22+.

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

=== "Rust"

    ```rust
    use apcore_a2a::{serve, APCoreA2AConfig, BackendSource};
    use std::path::PathBuf;

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        // 1. Point the backend at your extensions directory
        let source = BackendSource::ExtensionsDir(PathBuf::from("./extensions"));

        // 2. Launch the A2A server (the port lives in the bind URL)
        let config = APCoreA2AConfig {
            name: "My Agent".into(),
            url: "http://localhost:8000".into(),
            ..Default::default()
        };
        serve(source, config).await?; // blocks until shutdown
        Ok(())
    }
    ```

Your agent is now running. Verify by visiting:

=== "Python"

    ```
    http://localhost:8000/.well-known/agent-card.json
    ```

=== "TypeScript"

    ```
    http://localhost:8000/.well-known/agent-card.json
    ```

=== "Rust"

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

=== "Rust"

    ```rust
    use apcore::context::Context;
    use apcore::errors::ModuleError;
    use apcore::module::Module;
    use apcore::registry::registry::Registry;
    use apcore_a2a::{serve, APCoreA2AConfig, BackendSource};
    use async_trait::async_trait;
    use serde_json::{json, Value};
    use std::sync::Arc;

    struct Hello;

    #[async_trait]
    impl Module for Hello {
        fn input_schema(&self) -> Value {
            json!({ "type": "object",
                    "properties": { "name": { "type": "string" } },
                    "required": ["name"] })
        }

        fn output_schema(&self) -> Value {
            json!({ "type": "object",
                    "properties": { "result": { "type": "string" } } })
        }

        fn description(&self) -> &str { "Say hello to someone" }

        async fn execute(&self, inputs: Value, _ctx: &Context<Value>) -> Result<Value, ModuleError> {
            let name = inputs.get("name").and_then(Value::as_str).unwrap_or("world");
            Ok(json!({ "result": format!("Hello, {name}! Welcome to the A2A world.") }))
        }
    }

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let registry = Registry::new();
        registry.register_module("greeting.hello", Box::new(Hello))?;

        let config = APCoreA2AConfig {
            name: "Greeter Agent".into(),
            url: "http://localhost:8000".into(),
            ..Default::default()
        };
        serve(BackendSource::Registry(Arc::new(registry)), config).await?;
        Ok(())
    }
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
                {"role": "user", "parts": [{"text": "Hello from Python!"}]},
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
      { role: "user", parts: [{ text: "Hello from TypeScript!" }] },
      { metadata: { skillId: "greeting.hello" } },
    );
    const status = task["status"] as Record<string, unknown>;
    console.log(`Result: ${status["state"]}`);

    client.close();
    ```

=== "Rust"

    ```rust
    use apcore_a2a::A2AClient;
    use serde_json::json;

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = A2AClient::new("http://localhost:8000");

        // Discover the agent
        let card = client.discover().await?;
        let skills = card["skills"].as_array().map_or(0, |s| s.len());
        println!("Agent: {}, Skills: {}", card["name"], skills);

        // Send a message — structured inputs go in a `data` part;
        // `metadata.skillId` selects the module.
        let task = client
            .send_message(
                json!({
                    "messageId": "m1",
                    "role": "ROLE_USER",
                    "parts": [{ "data": { "name": "Rust" } }]
                }),
                Some(json!({ "skillId": "greeting.hello" })),
                None, // optional contextId
            )
            .await?;
        println!("Result: {}", task["status"]["state"]);
        Ok(())
    }
    ```

---

## Step 4: Streaming Responses

For long-running tasks, use SSE streaming to receive real-time updates.

=== "Python"

    ```python
    from apcore_a2a import A2AClient

    async with A2AClient("http://localhost:8000") as client:
        async for event in client.stream_message(
            {"role": "user", "parts": [{"text": "Process this data"}]},
            metadata={"skillId": "data.process"},
        ):
            # A2A 1.0 events are oneof-wrapped: statusUpdate / artifactUpdate
            if "statusUpdate" in event:
                print(f"Status: {event['statusUpdate']['status']['state']}")
            if "artifactUpdate" in event:
                print(f"Artifact: {event['artifactUpdate']['artifact']}")
    ```

=== "TypeScript"

    ```typescript
    import { A2AClient } from "apcore-a2a";

    const client = new A2AClient("http://localhost:8000");

    for await (const event of client.streamMessage(
      { role: "user", parts: [{ text: "Process this data" }] },
      { metadata: { skillId: "data.process" } },
    )) {
      // A2A 1.0 events are oneof-wrapped: statusUpdate / artifactUpdate
      if (event.statusUpdate) {
        console.log(`Status: ${event.statusUpdate.status.state}`);
      }
      if (event.artifactUpdate) {
        console.log(`Artifact:`, event.artifactUpdate.artifact);
      }
    }
    ```

=== "Rust"

    ```rust
    use apcore_a2a::A2AClient;
    use futures_util::StreamExt; // brings `.next()` to the event stream
    use serde_json::json;

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = A2AClient::new("http://localhost:8000");

        let mut stream = client.stream_message(
            json!({
                "messageId": "m1",
                "role": "ROLE_USER",
                "parts": [{ "text": "Process this data" }]
            }),
            Some(json!({ "skillId": "data.process" })),
            None,
        );

        // A2A 1.0 events are oneof-wrapped: statusUpdate / artifactUpdate.
        // The stream ends on a terminal TASK_STATE_* status.
        while let Some(event) = stream.next().await {
            let event = event?;
            if let Some(s) = event.get("statusUpdate") {
                println!("Status: {}", s["status"]["state"]);
            }
            if let Some(a) = event.get("artifactUpdate") {
                println!("Artifact: {}", a["artifact"]);
            }
        }
        Ok(())
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

=== "Rust"

    ```rust
    use apcore_a2a::{async_serve_with_auth, APCoreA2AConfig, BackendSource, JWTAuthenticator};
    use std::path::PathBuf;
    use std::sync::Arc;

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let source = BackendSource::ExtensionsDir(PathBuf::from("./extensions"));
        let config = APCoreA2AConfig {
            name: "Secure Agent".into(),
            url: "http://localhost:8000".into(),
            ..Default::default()
        };

        let auth = JWTAuthenticator::new(std::env::var("APCORE_JWT_SECRET")?);

        // Discovery and /health stay public; everything else requires a token.
        async_serve_with_auth(source, config, Some(Arc::new(auth))).await?;
        Ok(())
    }
    ```

    Client side — pass the full `Authorization` header value to `try_new`:

    ```rust
    use apcore_a2a::A2AClient;

    let client = A2AClient::try_new(
        "http://localhost:8000",
        Some("Bearer eyJ...".into()),
        None, // request timeout (default 30s)
        None, // Agent Card cache TTL
    )?;
    ```

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

=== "Rust"

    ```bash
    apcore-a2a --extensions-dir ./extensions --name "CLI Agent" --port 8000
    ```

    Available flags: `--extensions-dir` (`-e`), `--name` (`-n`), `--url`, and
    `--port` (`-p`). Run `apcore-a2a --help` for details.

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

=== "Rust (axum)"

    ```rust
    use apcore_a2a::{build_app, APCoreA2AConfig, BackendSource};
    use axum::Router;
    use std::path::PathBuf;

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let source = BackendSource::ExtensionsDir(PathBuf::from("./extensions"));
        let config = APCoreA2AConfig {
            url: "https://example.com/a2a".into(),
            ..Default::default()
        };

        // `build_app` returns the A2A router (and Agent Card) without serving.
        let (a2a_router, _card) = build_app(source, config).await?;

        let outer = Router::new().nest("/a2a", a2a_router);

        let listener = tokio::net::TcpListener::bind("0.0.0.0:8000").await?;
        axum::serve(listener, outer).await?;
        Ok(())
    }
    ```

---

## Configuration Reference

### `serve()` Options

| Option | Python | TypeScript | Rust | Default | Description |
|--------|--------|------------|------|---------|-------------|
| Host | `host` | `host` | (in `url`) | `"0.0.0.0"` (CLI: `"127.0.0.1"`) | Bind address |
| Port | `port` | `port` | (in `url`) | `8000` | Bind port |
| Name | `name` | `name` | `name` | From registry or `"Apcore Agent"` | Agent display name |
| Description | `description` | `description` | `description` | From registry | Agent description |
| Version | `version` | `version` | `version` | From registry | Semver version |
| URL | `url` | `url` | `url` | `http://{host}:{port}` | Public base URL (Rust derives the bind address from this) |
| Auth | `auth` | `auth` | `*_with_auth(…, auth)` | `None` | Authenticator instance |
| Storage | `task_store` | `taskStore` | `TaskStore` trait | In-memory | Task persistence backend |
| CORS | `cors_origins` | `corsOrigins` | `cors_origins` | `None` | Allowed CORS origins |
| Push | `push_notifications` | — | (built-in) | `false` | Enable webhook notifications (Python only) |
| Explorer | `explorer` | `explorer` | `explorer` | `false` | Enable Explorer UI |
| Metrics | `metrics` | `metrics` | — | `false` | Enable `/metrics` endpoint (Python/TS only) |
| Log Level | `log_level` | `logLevel` | — | `"info"` | Logging level |

!!! note "Rust configuration shape"
    Rust passes options as fields on `APCoreA2AConfig` (with `..Default::default()`)
    rather than keyword arguments, and there is no separate `host`/`port` — the bind
    address is parsed from `url`. Authentication is supplied through the
    `async_serve_with_auth` / `build_app_with_auth` entry points rather than a config
    field. The Rust crate does not serve a `/metrics` endpoint.

---

## What's Next?

- **Explorer UI**: Enable `explorer=True` to get an interactive browser UI at `/explorer` for testing your agent
- **Push Notifications**: Enable `push_notifications=True` for webhook-based async task updates
- **Multi-Agent Workflows**: Use `A2AClient` to orchestrate multiple agents in complex pipelines
- **Custom Storage**: Implement the `TaskStore` protocol for Redis/PostgreSQL persistence
- **Detailed Design**: See the [Technical Design](spec/tech-design.md) for an architecture deep-dive.
- **Feature Specs**: See the [Feature Specs Overview](features/overview.md) for implementation details.
