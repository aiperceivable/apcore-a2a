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

```python
# Top-level public exports:
from apcore_a2a._serve import serve, async_serve
from apcore_a2a.client import A2AClient

__all__ = ["serve", "async_serve", "A2AClient"]
```

---

## Component: `serve()` — `_serve.py`

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
    log_level: str | None = None,
) -> None:
    """Launch a compliant A2A agent server. Blocks until shutdown.

    Args:
        registry_or_executor: apcore Registry or Executor object (duck-typed).
        host: Bind address. Default: "0.0.0.0".
        port: Bind port. Default: 8000.
        name: Agent display name. Default: from registry config or "apcore-agent".
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
        execution_timeout: Seconds before task execution times out. Can also be set via A2A_EXECUTION_TIMEOUT environment variable. Default: 300.
        metrics: Enable GET /metrics endpoint. Default: False.
        log_level: Logging level ("debug","info","warning","error"). Default: "info".

    Raises:
        ValueError: registry_or_executor has zero modules, or invalid url.
        TypeError: auth or task_store do not satisfy their protocols.
    """
```

**Execution sequence:**

1. **Resolve** `registry_or_executor`:
   - Duck-type check: if object has `list()` and `get_definition()` → treat as Registry, wrap in Executor.
   - If object has `call_async()` → treat as Executor; extract Registry via `executor.registry`.
   - If neither → raise `TypeError("Expected apcore Registry or Executor")`.

2. **Validate** Registry has ≥ 1 module:
   ```python
   modules = registry.list()
   if not modules:
       raise ValueError("Registry contains zero modules; at least one module is required to serve an A2A agent")
   ```

3. **Resolve metadata** (fall through chain):
   - `name`: kwarg → `registry.config.get("project", {}).get("name")` → `"apcore-agent"`
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
) -> Starlette:
    """Build and return a configured Starlette ASGI app. Does NOT start uvicorn.

    Suitable for embedding in existing ASGI servers or test clients.

    Raises:
        ValueError: Zero modules in registry, or invalid url.
        TypeError: auth or task_store protocol violations.
    """
```

**Execution sequence:**

1. Apply steps 1–5 from `serve()` (resolve, validate, default).
2. Call `A2AServerFactory().create(registry, executor, ...)` to build `(app, agent_card)`.
3. Return the `Starlette` ASGI application.

**Use in testing:**
```python
from starlette.testclient import TestClient
app = await async_serve(registry)
client = TestClient(app)
```

**Use in FastAPI embedding:**
```python
from fastapi import FastAPI
outer = FastAPI()
a2a_app = await async_serve(registry, url="https://example.com/a2a")
outer.mount("/a2a", a2a_app)
```

---

## `A2AClient` Re-export

```python
# In apcore_a2a/__init__.py:
from apcore_a2a.client import A2AClient
```

Available at package level for convenience:
```python
from apcore_a2a import A2AClient

async with A2AClient("https://agent.example.com") as client:
    card = await client.discover()
```

---

## Typical Usage

### Minimal (development)
```python
import apcore_a2a
from my_modules import registry

apcore_a2a.serve(registry)
```

### With auth + push notifications
```python
from apcore_a2a import serve
from apcore_a2a.auth import JWTAuthenticator

auth = JWTAuthenticator(key=os.environ["JWT_SECRET"])
serve(registry, auth=auth, push_notifications=True, port=8080)
```

### ASGI embedding
```python
from apcore_a2a import async_serve

app = await async_serve(registry, url="https://myagent.example.com")
# app is a Starlette instance — pass to uvicorn, gunicorn, etc.
```

---

## File Structure

```
src/apcore_a2a/
    __init__.py      # exports: serve, async_serve, A2AClient
    _serve.py        # serve(), async_serve() implementations
```

The underscore prefix on `_serve.py` indicates internal; public surface is `__init__.py` only.

## Key Invariants

- `serve()` blocks; `async_serve()` returns an ASGI app — callers choose the deployment model
- Both functions apply identical validation (resolve → validate → default → protocol check)
- Protocol validation uses `isinstance()` against `runtime_checkable` protocols (duck-typing safe)
- `task_store=None` always defaults to `InMemoryTaskStore()` — server always has storage
- `A2AClient` re-exported at top level — no server imports pulled in by import
- Zero module registry raises `ValueError` immediately (fail fast before any server setup)

## Test Module

`tests/test_public_api.py`
