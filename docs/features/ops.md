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

```python
async def handle_health(self, request: Request) -> JSONResponse:
    uptime = time.monotonic() - self._started_at
    return JSONResponse({
        "status": "healthy",
        "module_count": len(self._registry.list()),
        "uptime_seconds": int(uptime),
        "version": self._version,
    })
```

HTTP 200:
```json
{
  "status": "healthy",
  "module_count": 5,
  "uptime_seconds": 3600,
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

Task state counters are maintained in `TransportManager` via callback from `TaskManager`:

```python
class TransportManager:
    def __init__(self, ...) -> None:
        self._counters = {
            "active": 0,
            "completed": 0,
            "failed": 0,
            "canceled": 0,
            "requests": 0,
        }
        self._started_at = time.monotonic()
```

`TaskManager` increments counters on `transition()` callbacks:
- `→ working` : `active += 1`
- `→ completed`: `active -= 1`, `completed += 1`
- `→ failed`   : `active -= 1`, `failed += 1`
- `→ canceled` : `active -= 1`, `canceled += 1`

`TransportManager.handle_jsonrpc()` increments `requests` on each valid JSON-RPC call.

### `serve()` kwarg

```python
def serve(
    registry_or_executor: object,
    *,
    metrics: bool = False,   # Enable GET /metrics endpoint
    ...
) -> None: ...
```

---

## Sub-feature 3: Dynamic Registration

**Implements**: FR-OPS-003, FR-AGC-005

**Purpose**: Allow modules to be added to the registry at runtime without restarting the server. The Agent Card is re-computed and the cached version is invalidated.

### API

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

### `AgentCardBuilder.invalidate_cache()`

```python
def invalidate_cache(self) -> None:
    """Invalidate cached Agent Cards. Called on module registration changes."""
    self._cached_card = None
    self._cached_extended_card = None
```

After invalidation, the next `GET /.well-known/agent.json` call triggers a full rebuild from the registry.

### Transport Hot-Swap

The `TransportManager` holds a reference to the agent card dict:
```python
class TransportManager:
    def update_agent_card(self, agent_card: dict) -> None:
        """Replace agent card. Thread-safe for async event loop."""
        self._agent_card = agent_card
        self._extended_card = None  # Reset extended card too
```

Since this is an async event loop context (single-threaded), the dict replacement is atomic.

---

## `serve()` kwargs Summary

| Kwarg | Type | Default | Description |
|-------|------|---------|-------------|
| `metrics` | bool | `False` | Enable `GET /metrics` endpoint |

The other ops features (health, dynamic registration) are always available when the server is running.

---

## File Structure

Ops features are implemented across existing files:

```
src/apcore_a2a/server/
    factory.py      # _build_health_handler(), _build_metrics_handler(), register_module()
src/apcore_a2a/adapters/
    agent_card.py   # invalidate_cache()
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
