---
description: "Ops module spec: operational endpoints GET /health (liveness) and GET /metrics (task counters) plus dynamic Agent Card re-registration on runtime registry changes."
---

# Feature: Ops Module

| Field | Value |
|-------|-------|
| Feature ID | F-11 |
| Name | ops |
| Priority | P2 |
| SRS Refs | FR-OPS-001, FR-OPS-002, FR-OPS-003, FR-AGC-005 |
| Tech Design | §4.4.4 TransportManager (health/metrics), §4.3.1 AgentCardBuilder (dynamic registration) |
| Depends On | F-03 (server-core — TransportManager, A2AServerFactory) |
| Blocks | None |

## Purpose

Operational endpoints and dynamic module registration. Three sub-features:

1. **Health** (`GET /health`) — liveness probe for container orchestration.
2. **Metrics** (`GET /metrics`) — lightweight task counters for monitoring.
3. **Dynamic Registration** — hot-reload of Agent Card when registry changes at runtime.

---

## Sub-feature 1: Health Endpoint

**Implemented in**: `_build_health_handler()` (`server/factory.py`)

**Route**: `GET /health`
**Auth**: Not required (always exempt)

### Response — Healthy

=== "Python"

    ```python
    async def handle_health(self, request: Request) -> JSONResponse:
        uptime = time.monotonic() - self._started_at
        return JSONResponse({
            "status": "healthy",
            "module_count": len(self._registry.list()),
            "uptime_seconds": uptime,  # float (seconds since start)
            "version": self._version,
        })
    ```

=== "TypeScript"

    ```typescript
    // src/server/factory.ts — GET /health
    // → { status, uptimeSeconds, moduleCount, version }
    // 503 with { status: "unhealthy", reason } when the task store is unreachable.
    app.get("/health", (_req, res) => {
      res.json({
        status: "healthy",
        moduleCount: registry.list().length,
        uptimeSeconds: (Date.now() - startedAt) / 1000,
        version: VERSION,
      });
    });
    ```

=== "Rust"

    ```rust
    // src/server/factory.rs — GET /health is always public.
    // Returns { "status": "healthy" } (503 with a reason when unhealthy).
    async fn handle_health() -> Json<Value> {
        Json(json!({ "status": "healthy" }))
    }
    ```

HTTP 200:
```json
{
  "status": "healthy",
  "module_count": 5,
  "uptime_seconds": 3600.5,
  "version": "1.0.0"
}
```

### Response — Unhealthy

When task store is unreachable (e.g., custom store raises on `get()`):
```json
{
  "status": "unhealthy",
  "reason": "Task store unavailable"
}
```
HTTP 503.

### Health Check Probe

The health response is designed for container liveness probes:
```yaml
# Kubernetes liveness probe
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## Sub-feature 2: Metrics Endpoint

> **SDK availability:** `/metrics` is implemented in the **Python and TypeScript SDKs only**.
> The Rust crate does not serve a `/metrics` endpoint (its `metrics` config flag is inert);
> SSE streaming and push notifications are fully supported in Rust.

**Implemented in**: `_build_metrics_handler()` (`server/factory.py`)

**Route**: `GET /metrics`
**Auth**: Not required
**Enabled by**: `metrics=True` in `serve()` kwargs (default: `False`)

### Response

HTTP 200:
```json
{
  "active_tasks": 3,
  "completed_tasks": 147,
  "failed_tasks": 2,
  "canceled_tasks": 1,
  "input_required_tasks": 0,
  "total_requests": 512,
  "uptime_seconds": 3600
}
```

HTTP 404 (when `metrics=False`, the default):
```json
{"detail": "Not Found"}
```

### Counter Implementation

Task state counters are maintained in a `_MetricsState` dataclass within `server/factory.py`, updated via `on_state_change` callback from `ApCoreAgentExecutor`:

=== "Python"

    ```python
    @dataclass
    class _MetricsState:
        active: int = 0
        completed: int = 0
        failed: int = 0
        canceled: int = 0
        input_required: int = 0
        started_at: float = field(default_factory=time.monotonic)
    ```

=== "TypeScript"

    ```typescript
    // src/server/factory.ts — counters updated via the onStateChange callback
    // from ApCoreAgentExecutor; served by GET /metrics when metrics: true.
    interface MetricsState {
      active: number;
      completed: number;
      failed: number;
      canceled: number;
      inputRequired: number;
      startedAt: number;
    }
    ```

=== "Rust"

    > Not applicable — the Rust crate does not serve a `/metrics` endpoint (the `metrics` flag is inert), so no metrics counters are maintained.

State change callbacks update counters:
- `→ working` : `active += 1`
- `→ completed`: `active -= 1`, `completed += 1`
- `→ failed`   : `active -= 1`, `failed += 1`
- `→ canceled` : `active -= 1`, `canceled += 1`
- `→ input_required`: `input_required += 1`

### `serve()` option

=== "Python"

    ```python
    def serve(
        registry_or_executor: object,
        *,
        metrics: bool = False,   # Enable GET /metrics endpoint
        ...
    ) -> None: ...
    ```

=== "TypeScript"

    ```typescript
    import { serve } from "apcore-a2a";
    // metrics: true enables the GET /metrics endpoint (default: false)
    serve(registry, { metrics: true, port: 8000 });
    ```

=== "Rust"

    > Not applicable — `APCoreA2AConfig.metrics` exists for parity but is inert; the Rust crate does not serve a `/metrics` endpoint.

---

## Sub-feature 3: Dynamic Registration

**Implements**: FR-OPS-003, FR-AGC-005

**Purpose**: Allow modules to be added to the registry at runtime without restarting the server. The Agent Card is re-computed and the cached version is invalidated.

### API

=== "Python"

    ```python
    class A2AServerFactory:
        def register_module(self, module_id: str, descriptor: object) -> None:
            """Register a new module at runtime.

            Steps:
            1. Add module to registry via registry.register(module_id, descriptor).
            2. Invalidate cached Agent Card: agent_card_builder.invalidate_cache() (card rebuilt lazily on next request).
            3. Log INFO: "Dynamically registered module: {module_id}".

            Raises:
                ValueError: If module_id is already registered.
                TypeError: If descriptor is not a valid ModuleDescriptor.
            """
    ```

=== "TypeScript"

    ```typescript
    // src/server/factory.ts — registers a module at runtime and
    // invalidates the cached Agent Card so it is rebuilt on the next request.
    class A2AServerFactory {
      registerModule(moduleId: string, descriptor: unknown): void;
    }
    ```

=== "Rust"

    > No runtime `register_module` on the factory. Register modules on the `Registry`
    > before serving, then pass it via `BackendSource::Registry(...)`:

    ```rust
    let registry = Registry::new();
    registry.register_module("greeting.hello", Box::new(Hello))?;
    serve(BackendSource::Registry(Arc::new(registry)), APCoreA2AConfig::default()).await?;
    ```

### `AgentCardBuilder.invalidate_cache()`

=== "Python"

    ```python
    def invalidate_cache(self) -> None:
        """Invalidate cached Agent Cards. Called on module registration changes."""
        self._cached_card = None
        self._cached_extended_card = None
    ```

=== "TypeScript"

    ```typescript
    // src/adapters/agent-card.ts — clears the cached card so it is rebuilt lazily.
    class AgentCardBuilder {
      invalidateCache(): void;
    }
    ```

=== "Rust"

    ```rust
    // src/adapters/agent_card.rs — clears the cached card so it is rebuilt lazily.
    impl AgentCardBuilder {
        pub fn invalidate_cache(&self) { /* ... */ }
    }
    ```

After invalidation, the next `GET /.well-known/agent-card.json` call triggers a full rebuild from the registry.

### Agent Card Hot-Swap

Agent Card updates are handled via `AgentCardBuilder.invalidate_cache()` combined with a
`card_modifier` callback passed to `A2AStarletteApplication`. The a2a-sdk lazily rebuilds
the card on the next request after cache invalidation. There is no `TransportManager` —
transport is handled by a2a-sdk's `A2AStarletteApplication`.

---

## `serve()` kwargs Summary

| Kwarg | Type | Default | Description |
|-------|------|---------|-------------|
| `metrics` | bool | `False` | Enable `GET /metrics` endpoint |

The other ops features (health, dynamic registration) are always available when the server is running.

---

## File Structure

Ops features are implemented across existing files:

=== "Python"

    ```
    src/apcore_a2a/server/
        factory.py      # _build_health_handler(), _build_metrics_handler(), register_module()
    src/apcore_a2a/adapters/
        agent_card.py   # invalidate_cache()
    ```

=== "TypeScript"

    ```
    src/server/
        factory.ts      # GET /health, GET /metrics, registerModule()
    src/adapters/
        agent-card.ts   # invalidateCache()
    ```

=== "Rust"

    ```
    src/server/
        factory.rs      # GET /health (no /metrics; metrics flag inert)
    src/adapters/
        agent_card.rs   # invalidate_cache()
    ```

No new files required.

---

## Key Invariants

- `GET /health` is **always exempt** from auth — never returns 401
- `GET /metrics` returns 404 by default; returns 200 only when `metrics=True`
- Health check checks task store reachability — returns 503 if store is unavailable
- Dynamic registration invalidates Agent Card cache immediately
- Agent Card rebuild is synchronous and completes before `register_module()` returns
- `GET /health` and `GET /metrics` paths are always in `AuthMiddleware.exempt_paths`

## Test Module

`tests/server/test_ops.py`
