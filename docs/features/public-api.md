---
description: "Public API spec: the top-level package entry point exposing serve() (blocking), async_serve() (ASGI app), and re-exporting A2AClient, wiring all subsystems via A2AServerFactory."
---

# Feature: Public API

| Field | Value |
|-------|-------|
| Feature ID | F-08 |
| Name | public-api |
| Priority | P0 |
| SRS Refs | FR-SRV-001, FR-SRV-002, FR-SRV-003 |
| Tech Design | §4.2 Public API Layer |
| Depends On | F-01 (adapters), F-02 (storage), F-03 (server-core), F-04 (streaming), F-05 (push), F-06 (auth), F-07 (client) |
| Blocks | None (top-level entry point) |

## Purpose

Top-level package entry point. Exposes `serve()` (blocking), `async_serve()` (returns ASGI app), and re-exports `A2AClient`. Mirrors apcore-mcp's `serve(registry)` pattern so module developers have a familiar one-call API. Wires all subsystems together through `A2AServerFactory`.

## File: `__init__.py`

=== "Python"

    ```python
    # Top-level public exports:
    from apcore_a2a._serve import serve, async_serve
    from apcore_a2a.client import A2AClient

    __all__ = [
        "serve",
        "async_serve",
        "A2AClient",
        "__version__",
        # Re-exported from submodules for convenience:
        "Authenticator",
        "JWTAuthenticator",
        "ClaimMapping",
        "AuthMiddleware",
        "auth_identity_var",
        "AgentCardBuilder",
        "SkillMapper",
        "SchemaConverter",
        "ErrorMapper",
        "PartConverter",
        "A2AServerFactory",
        "ApCoreAgentExecutor",
    ]
    ```

=== "TypeScript"

    ```typescript
    // src/index.ts — top-level public exports:
    export { VERSION } from "./version.js";
    export { serve, asyncServe } from "./serve.js";
    export { A2AClient } from "./client/client.js";

    // Re-exported from submodules for convenience:
    export type { Authenticator, ClaimMapping } from "./auth/types.js";
    export { JWTAuthenticator } from "./auth/jwt.js";
    export { createAuthMiddleware } from "./auth/middleware.js";
    export { authIdentityStore, getAuthIdentity } from "./auth/storage.js";
    export {
      AgentCardBuilder,
      SkillMapper,
      SchemaConverter,
      ErrorMapper,
      PartConverter,
    } from "./adapters/index.js";
    export { A2AServerFactory, ApCoreAgentExecutor } from "./server/index.js";
    ```

=== "Rust"

    ```rust
    // src/lib.rs — top-level public re-exports:
    pub const VERSION: &str = "0.4.0";

    pub use apcore_a2a::{
        serve, async_serve, async_serve_with_auth, build_app, build_app_with_auth,
        APCoreA2A, APCoreA2ABuilder, APCoreA2AConfig, APCoreA2AError, BackendSource,
    };
    pub use auth::{
        Authenticator, ClaimMapping, JWTAuthenticator,
        AuthMiddlewareLayer, AuthMiddlewareService, AUTH_IDENTITY,
    };
    pub use adapters::{
        register_a2a_error_formatter, A2aErrorFormatter, AdapterError,
        AgentCardBuilder, ErrorMapper, PartConverter, SchemaConverter, SkillMapper,
    };
    pub use server::{A2AServerFactory, ApCoreAgentExecutor};
    pub use client::{A2AClient, A2AClientError, AgentCardFetcher, ClientResult};
    ```

---

## Component: `serve()` — `_serve.py`

=== "Python"

    ```python
    def serve(
        registry_or_executor: object,
        *,
        host: str = "0.0.0.0",
        port: int = 8000,
        name: str | None = None,
        description: str | None = None,
        version: str | None = None,
        url: str | None = None,
        auth: object | None = None,
        task_store: object | None = None,
        cors_origins: list[str] | None = None,
        push_notifications: bool = False,
        explorer: bool = False,
        explorer_prefix: str = "/explorer",
        cancel_on_disconnect: bool = True,
        shutdown_timeout: int = 30,
        execution_timeout: int = 300,
        metrics: bool = False,
        sys_modules: bool = False,
        log_level: str | None = None,
    ) -> None:
        """Launch a compliant A2A agent server. Blocks until shutdown.

        Args:
            registry_or_executor: apcore Registry or Executor object (duck-typed).
            host: Bind address. Default: "0.0.0.0".
            port: Bind port. Default: 8000.
            name: Agent display name. Default: from registry config or "Apcore Agent".
            description: Agent description. Default: from registry config.
            version: Semver version. Default: from registry config or "0.0.0".
            url: Public base URL for Agent Card. Default: f"http://{host}:{port}".
            auth: Authenticator object (duck-typed). Default: None (no auth).
            task_store: TaskStore object (duck-typed). Default: InMemoryTaskStore().
            cors_origins: List of allowed CORS origins. Default: None (no CORS).
            push_notifications: Enable webhook push notifications. Default: False.
            explorer: Enable Explorer UI at explorer_prefix. Default: False.
            explorer_prefix: URL prefix for Explorer. Default: "/explorer".
            cancel_on_disconnect: Deprecated. Has no effect; DefaultRequestHandler does not support disabling cancel-on-disconnect. Default: True.
            shutdown_timeout: Seconds to wait for graceful shutdown. Default: 30.
            execution_timeout: Seconds before task execution times out. Can also be set via APCORE_A2A_EXECUTION_TIMEOUT environment variable. Default: 300.
            metrics: Enable GET /metrics endpoint. Default: False.
            sys_modules: Register apcore sys.* modules (health/usage/manifest) and
                wire observability middleware. Default: False.
            log_level: Logging level ("debug","info","warning","error"). Default: "info".

        Raises:
            ValueError: registry_or_executor has zero modules, or invalid url.
            TypeError: auth or task_store do not satisfy their protocols.
        """
    ```

=== "TypeScript"

    ```typescript
    // src/serve.ts — serve() blocks: starts http.createServer(app),
    // graceful SIGTERM/SIGINT. Returns void.
    export function serve(registryOrExecutor: unknown, opts?: ServeOptions): void

    export interface ServeOptions extends AsyncServeOptions {
      host?: string;            // "0.0.0.0"
      port?: number;            // 8000
      logLevel?: string;        // "info"
      shutdownTimeout?: number; // seconds
    }
    export interface AsyncServeOptions {
      name?: string;
      description?: string;
      version?: string;
      url?: string;
      auth?: Authenticator;
      taskStore?: TaskStore;
      corsOrigins?: string[];
      pushNotifications?: boolean;
      explorer?: boolean;
      explorerPrefix?: string;
      executionTimeout?: number; // seconds (300)
      metrics?: boolean;
      sysModules?: boolean;
    }
    ```

=== "Rust"

    ```rust
    // src/apcore_a2a.rs — serve() blocks until shutdown.
    // Rust folds host+port into config.url (no host/port fields).
    // Auth is passed via async_serve_with_auth, not a config field.
    pub async fn serve(
        source: BackendSource,
        config: APCoreA2AConfig,
    ) -> Result<(), APCoreA2AError>

    pub struct APCoreA2AConfig {
        pub name: String,              // "apcore-a2a"
        pub description: String,       // "apcore A2A agent"
        pub version: String,           // VERSION
        pub url: String,               // "http://localhost:8000" (bind addr derived from this)
        pub execution_timeout: u64,    // 300
        pub explorer: bool,            // false
        pub metrics: bool,             // false (INERT in Rust)
        pub sys_modules: bool,         // false
        pub cors_origins: Vec<String>, // []
    }

    pub enum BackendSource {
        ExtensionsDir(PathBuf),
        Registry(Arc<Registry>),
        Executor(Arc<Executor>),
    }
    pub enum APCoreA2AError { EmptyRegistry, Server(String) }
    ```

**Execution sequence:**

1. **Resolve** `registry_or_executor`:
   - If `str` or `pathlib.Path` → treat as extensions directory path; auto-create Registry, discover modules, build Executor.
   - Duck-type check: if object has `call_async()` → treat as Executor; extract Registry via `executor.registry`.
   - If object has `list()` and `get_definition()` → treat as Registry.
   - If neither → raise `TypeError("Expected apcore Registry, Executor, or path string")`.

2. **Validate** Registry has ≥ 1 module:
   ```python
   modules = registry.list()
   if not modules:
       raise ValueError("Registry contains zero modules; at least one module is required to serve an A2A agent")
   ```

3. **Resolve metadata** (fall through chain):
   - `name`: kwarg → `registry.config.get("project", {}).get("name")` → `"Apcore Agent"`
   - `version`: kwarg → `registry.config.get("project", {}).get("version")` → `"0.0.0"`
   - `description`: kwarg → `registry.config.get("project", {}).get("description")` → `f"apcore agent with {len(modules)} skills"`
   - `url`: kwarg → `f"http://{host}:{port}"`

4. **Default** `task_store` to `InMemoryTaskStore()` if `None`.

5. **Protocol validation**:
   ```python
   if auth is not None and not isinstance(auth, Authenticator):
       missing = [m for m in ("authenticate", "security_schemes") if not hasattr(auth, m)]
       raise TypeError(f"auth missing required methods: {missing}")
   missing_store = [m for m in ("save", "get", "delete") if not hasattr(task_store, m)]
   if missing_store:
       raise TypeError(f"task_store missing required methods: {missing_store}")
   ```

6. **Configure logging** if `log_level` provided:
   ```python
   logging.basicConfig(level=getattr(logging, log_level.upper(), logging.INFO))
   ```

7. **Delegate** to `async_serve()` internals to build the ASGI app.

8. **Configure signal handlers** for `SIGINT` / `SIGTERM`:
   - On signal: trigger `uvicorn.Server.handle_exit()` for graceful shutdown.
   - Wait up to `shutdown_timeout` seconds for in-flight requests to drain.

9. **Start** `uvicorn.Server(config)` and block until server exits.

---

## Component: `async_serve()` — `_serve.py`

=== "Python"

    ```python
    async def async_serve(
        registry_or_executor: object,
        *,
        name: str | None = None,
        description: str | None = None,
        version: str | None = None,
        url: str = "http://localhost:8000",
        auth: object | None = None,
        task_store: object | None = None,
        cors_origins: list[str] | None = None,
        push_notifications: bool = False,
        explorer: bool = False,
        explorer_prefix: str = "/explorer",
        cancel_on_disconnect: bool = True,
        execution_timeout: int = 300,
        metrics: bool = False,
        sys_modules: bool = False,
    ) -> Starlette:
        """Build and return a configured Starlette ASGI app. Does NOT start uvicorn.

        Suitable for embedding in existing ASGI servers or test clients.

        Raises:
            ValueError: Zero modules in registry, or invalid url.
            TypeError: auth or task_store protocol violations.
        """
    ```

=== "TypeScript"

    ```typescript
    // src/serve.ts — asyncServe() does NOT start a server; it returns the
    // Express app, ready to mount (outer.use("/a2a", app)) or hand to supertest.
    export async function asyncServe(
        registryOrExecutor: unknown,
        opts?: AsyncServeOptions,
    ): Promise<Express>
    ```

=== "Rust"

    ```rust
    // src/apcore_a2a.rs — async_serve() runs the server; build_app() does NOT
    // start it, returning the axum (Router, AgentCard) for embedding/testing.
    pub async fn async_serve(
        source: BackendSource,
        config: APCoreA2AConfig,
    ) -> Result<(), APCoreA2AError>

    pub async fn async_serve_with_auth(
        source: BackendSource,
        config: APCoreA2AConfig,
        auth: Option<Arc<dyn Authenticator>>,
    ) -> Result<(), APCoreA2AError>

    pub async fn build_app(
        source: BackendSource,
        config: APCoreA2AConfig,
    ) -> Result<(Router, AgentCard), APCoreA2AError>

    pub async fn build_app_with_auth(
        source: BackendSource,
        config: APCoreA2AConfig,
        auth: Option<Arc<dyn Authenticator>>,
    ) -> Result<(Router, AgentCard), APCoreA2AError>
    ```

**Execution sequence:**

1. Apply steps 1–5 from `serve()` (resolve, validate, default).
2. Call `A2AServerFactory().create(registry, executor, ...)` to build `(app, agent_card)`.
3. Return the `Starlette` ASGI application.

**Use in testing:**

=== "Python"

    ```python
    from starlette.testclient import TestClient
    app = await async_serve(registry)
    client = TestClient(app)
    ```

=== "TypeScript"

    ```typescript
    import request from "supertest";
    const app = await asyncServe(registry);
    const res = await request(app).get("/.well-known/agent-card.json");
    ```

=== "Rust"

    ```rust
    // build_app() returns an axum Router you can drive with tower's
    // oneshot test harness (no network needed).
    use apcore_a2a::{build_app, APCoreA2AConfig, BackendSource};
    use tower::ServiceExt; // for `oneshot`
    use std::path::PathBuf;

    let (app, _card) = build_app(
        BackendSource::ExtensionsDir(PathBuf::from("./extensions")),
        APCoreA2AConfig::default(),
    ).await?;
    let response = app
        .oneshot(http::Request::get("/.well-known/agent-card.json").body(axum::body::Body::empty())?)
        .await?;
    ```

**Use in host-app embedding:**

=== "Python"

    ```python
    from fastapi import FastAPI
    outer = FastAPI()
    a2a_app = await async_serve(registry, url="https://example.com/a2a")
    outer.mount("/a2a", a2a_app)
    ```

=== "TypeScript"

    ```typescript
    import express from "express";
    import { asyncServe } from "apcore-a2a";
    const outer = express();
    const a2aApp = await asyncServe(registry, { url: "https://example.com/a2a" });
    outer.use("/a2a", a2aApp);
    outer.listen(8000);
    ```

=== "Rust"

    ```rust
    use apcore_a2a::{build_app, APCoreA2AConfig, BackendSource};
    use axum::Router;
    use std::path::PathBuf;

    let source = BackendSource::ExtensionsDir(PathBuf::from("./extensions"));
    let config = APCoreA2AConfig { url: "https://example.com/a2a".into(), ..Default::default() };
    let (a2a_router, _card) = build_app(source, config).await?;
    let outer = Router::new().nest("/a2a", a2a_router);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8000").await?;
    axum::serve(listener, outer).await?;
    ```

---

## `A2AClient` Re-export

=== "Python"

    ```python
    # In apcore_a2a/__init__.py:
    from apcore_a2a.client import A2AClient
    ```

=== "TypeScript"

    ```typescript
    // In src/index.ts:
    export { A2AClient } from "./client/client.js";
    ```

=== "Rust"

    ```rust
    // In src/lib.rs:
    pub use client::A2AClient;
    ```

Available at package level for convenience:

=== "Python"

    ```python
    from apcore_a2a import A2AClient

    async with A2AClient("https://agent.example.com") as client:
        card = await client.discover()
    ```

=== "TypeScript"

    ```typescript
    import { A2AClient } from "apcore-a2a";

    const client = new A2AClient("https://agent.example.com");
    const card = await client.discover();
    client.close();
    ```

=== "Rust"

    ```rust
    use apcore_a2a::A2AClient;

    let client = A2AClient::new("https://agent.example.com");
    let card = client.discover().await?;
    client.close().await;
    ```

---

## Typical Usage

### Minimal (development)

=== "Python"

    ```python
    import apcore_a2a
    from my_modules import registry

    apcore_a2a.serve(registry)
    ```

=== "TypeScript"

    ```typescript
    import { serve } from "apcore-a2a";
    serve(registry, { name: "My Agent", port: 8000 });
    ```

=== "Rust"

    ```rust
    use apcore_a2a::{serve, APCoreA2AConfig, BackendSource};
    use std::path::PathBuf;

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let source = BackendSource::ExtensionsDir(PathBuf::from("./extensions"));
        let config = APCoreA2AConfig { name: "My Agent".into(), url: "http://localhost:8000".into(), ..Default::default() };
        serve(source, config).await?;
        Ok(())
    }
    ```

### With auth + push notifications

=== "Python"

    ```python
    from apcore_a2a import serve
    from apcore_a2a.auth import JWTAuthenticator

    auth = JWTAuthenticator(key=os.environ["APCORE_JWT_SECRET"])
    serve(registry, auth=auth, push_notifications=True, port=8080)
    ```

=== "TypeScript"

    ```typescript
    import { serve, JWTAuthenticator } from "apcore-a2a";

    const auth = new JWTAuthenticator(process.env.APCORE_JWT_SECRET!);
    serve(registry, { auth, pushNotifications: true, port: 8080 });
    ```

=== "Rust"

    ```rust
    // Rust passes auth via async_serve_with_auth (no auth config field);
    // push-notification routes are always served.
    use apcore_a2a::{async_serve_with_auth, APCoreA2AConfig, BackendSource, JWTAuthenticator};
    use std::{path::PathBuf, sync::Arc};

    let source = BackendSource::ExtensionsDir(PathBuf::from("./extensions"));
    let config = APCoreA2AConfig { url: "http://localhost:8080".into(), ..Default::default() };
    let auth = JWTAuthenticator::new(std::env::var("APCORE_JWT_SECRET")?);
    async_serve_with_auth(source, config, Some(Arc::new(auth))).await?;
    ```

### App embedding

=== "Python"

    ```python
    from apcore_a2a import async_serve

    app = await async_serve(registry, url="https://myagent.example.com")
    # app is a Starlette instance — pass to uvicorn, gunicorn, etc.
    ```

=== "TypeScript"

    ```typescript
    import { asyncServe } from "apcore-a2a";

    const app = await asyncServe(registry, { url: "https://myagent.example.com" });
    // app is an Express instance — mount it or pass to http.createServer().
    ```

=== "Rust"

    ```rust
    use apcore_a2a::{build_app, APCoreA2AConfig, BackendSource};
    use std::path::PathBuf;

    let source = BackendSource::ExtensionsDir(PathBuf::from("./extensions"));
    let config = APCoreA2AConfig { url: "https://myagent.example.com".into(), ..Default::default() };
    // build_app returns an axum (Router, AgentCard) — nest it or pass to axum::serve.
    let (app, _card) = build_app(source, config).await?;
    ```

---

## File Structure

=== "Python"

    ```
    src/apcore_a2a/
        __init__.py      # exports: serve, async_serve, A2AClient, __version__,
                         #          + re-exports from auth, adapters, server submodules
        _serve.py        # serve(), async_serve() implementations
    ```

    The underscore prefix on `_serve.py` indicates internal; public surface is `__init__.py` only.

=== "TypeScript"

    ```
    src/
        index.ts         # exports: VERSION, serve, asyncServe, A2AClient,
                         #          + re-exports from auth, adapters, server submodules
        serve.ts         # serve(), asyncServe() implementations
    ```

    Public surface is `src/index.ts`; `src/serve.ts` holds the implementations.

=== "Rust"

    ```
    src/
        lib.rs           # pub use: VERSION, serve, async_serve, async_serve_with_auth,
                         #          build_app(_with_auth), A2AClient,
                         #          + re-exports from auth, adapters, server modules
        apcore_a2a.rs    # serve(), async_serve(), build_app() implementations
    ```

    Public surface is `src/lib.rs`; `src/apcore_a2a.rs` holds the implementations.

## Key Invariants

- `serve()` blocks; `async_serve()` returns an ASGI app — callers choose the deployment model
- Both functions apply identical validation (resolve → validate → default → protocol check)
- Protocol validation uses `isinstance()` against `runtime_checkable` protocols (duck-typing safe)
- `task_store=None` always defaults to `InMemoryTaskStore()` — server always has storage
- `A2AClient` re-exported at top level — no server imports pulled in by import
- Zero module registry raises `ValueError` immediately (fail fast before any server setup)

## Test Module

`tests/test_public_api.py`
