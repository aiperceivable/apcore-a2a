# Feature: Server Core

| Field | Value |
|-------|-------|
| Feature ID | F-03 |
| Name | server-core |
| Priority | P0 |
| SRS Refs | FR-SRV-001..005, FR-TSK-001..006, FR-EXE-001..003, FR-MSG-001, FR-AGC-003 |
| Tech Design | §4.4.1..4.4.4 |
| Depends On | F-01 (adapters), F-02 (storage) |
| Blocks | F-04 (streaming), F-05 (push), F-08 (public-api) |

> **Implementation Note**: The `ExecutionRouter`, `TaskManager`, and `TransportManager` classes
> described in this spec have been superseded by `ApCoreAgentExecutor` (implements
> `a2a.server.agent_execution.AgentExecutor`) and `A2AServerFactory` (uses
> `a2a.server.apps.jsonrpc.starlette_app.A2AStarletteApplication` +
> `a2a.server.request_handlers.default_request_handler.DefaultRequestHandler`). This document
> reflects the original design intent.

## Purpose

Core server machinery: factory that wires everything together, router that dispatches A2A messages to apcore Executor, task manager with the state machine, and transport that serves JSON-RPC + REST endpoints.

## Components

### 1. `A2AServerFactory` — `server/factory.py`

Creates the full ASGI application with all routes configured.

```python
class A2AServerFactory:
    def __init__(self) -> None:
        self._skill_mapper = SkillMapper()
        self._schema_converter = SchemaConverter()
        self._agent_card_builder = AgentCardBuilder(self._skill_mapper)
        self._error_mapper = ErrorMapper()
        self._part_converter = PartConverter(self._schema_converter)

    def create(
        self,
        registry: object,
        executor: object,
        *,
        name: str,
        description: str,
        version: str,
        url: str,
        task_store: TaskStore,
        auth: Authenticator | None = None,
        push_notifications: bool = False,
        cancel_on_disconnect: bool = True,
        execution_timeout: int = 300,
        cors_origins: list[str] | None = None,
        explorer: bool = False,
        explorer_prefix: str = "/explorer",
    ) -> tuple[Starlette, dict]:
        """Build ASGI app and Agent Card. Returns (app, agent_card)."""
```

**Creation sequence:**
1. Build skills via `_skill_mapper` from all registry modules.
2. Determine capabilities (streaming support, push, history).
3. Build Agent Card via `_agent_card_builder.build(...)`.
4. Create `TaskManager(task_store)`.
5. Create `ExecutionRouter(executor, task_manager, part_converter, error_mapper, execution_timeout)`.
6. Create `StreamingHandler(task_manager, router)`.
7. If `push_notifications`: create `PushNotificationManager(task_store)`.
8. Build Starlette routes:
   - `GET /.well-known/agent.json` → Agent Card (no auth)
   - `POST /` → JSON-RPC dispatch
   - `GET /health` → health check
   - `GET /agent/authenticatedExtendedCard` → extended card (if auth)
9. Apply `AuthMiddleware` if auth provided. Exempt: `{"/.well-known/agent.json", "/health", "/metrics"}`.
10. Apply CORS middleware if `cors_origins` provided.
11. Mount Explorer at `explorer_prefix` if `explorer=True`.

---

### 2. `ExecutionRouter` — `server/router.py`

Routes A2A message handling to apcore Executor.

```python
class ExecutionRouter:
    def __init__(
        self,
        executor: object,
        task_manager: TaskManager,
        part_converter: PartConverter,
        error_mapper: ErrorMapper,
        execution_timeout: int = 300,
    ) -> None: ...

    async def handle_message_send(
        self,
        params: dict,
        identity: object | None = None,
    ) -> dict:
        """Synchronous execution. Returns completed Task dict."""

    async def handle_message_stream(
        self,
        params: dict,
        task_id: str,
        identity: object | None = None,
    ) -> AsyncGenerator[dict, None]:
        """Streaming execution. Yields event dicts for StreamingHandler."""

    async def handle_resume(
        self,
        params: dict,
        task: dict,
        identity: object | None = None,
    ) -> dict:
        """Resume input_required task with follow-up message."""

    def _check_accepts_context(self, method) -> bool:
        """Inspect method signature for 'context' parameter."""
```

**`handle_message_send` steps:**
1. Extract `skill_id = params.get("message", {}).get("metadata", {}).get("skillId")`.
   - Missing → return error `{"code": -32602, "message": "Missing required parameter: metadata.skillId"}`.
2. Validate skill_id is a known module → error -32601 if not.
3. `task = await task_manager.create_task(context_id=params.get("message", {}).get("contextId"))`.
4. Parse Parts to input via `part_converter.parts_to_input(parts, descriptor)` → error -32602 on failure.
5. `await task_manager.transition(task["id"], "working")`.
6. Build apcore `Context` with identity (if provided via `auth_identity_var`).
7. `output = await asyncio.wait_for(executor.call_async(skill_id, inputs, context), execution_timeout)`.
8. On `ApprovalPendingError` → transition to `input_required`, return Task.
9. On any other exception → transition to `failed`, map error, return Task.
10. On success → `parts = part_converter.output_to_parts(output)`, transition to `completed` with artifacts, return Task.

**Identity bridging:**
```python
from apcore_a2a.auth.middleware import auth_identity_var

async def handle_message_send(self, params, identity=None):
    identity = identity or auth_identity_var.get()
    context = Context.create(identity=identity) if identity else Context.create()
```

**Streaming execution** (`handle_message_stream`):
1. Detect if executor has `stream()` via `hasattr(executor, "stream")`.
2. If yes: iterate `executor.stream(skill_id, inputs, context)` → yield `TaskArtifactUpdateEvent` per chunk.
3. If no: call `call_async()` and yield single artifact on completion.
4. Always yield `TaskStatusUpdateEvent` on state transitions.

---

### 3. `TaskManager` — `server/task_manager.py`

Task lifecycle state machine with per-task asyncio locks and event pub/sub.

```python
class TaskManager:
    _TRANSITIONS: dict[str, set[str]] = {
        "submitted":      {"working", "canceled", "failed"},
        "working":        {"completed", "failed", "canceled", "input_required"},
        "input_required": {"working", "canceled", "failed"},
        "completed":      set(),   # terminal — no further transitions
        "failed":         set(),   # terminal
        "canceled":       set(),   # terminal
    }

    def __init__(self, store: TaskStore, *, enable_history: bool = True) -> None: ...

    async def create_task(
        self,
        context_id: str | None = None,
        metadata: dict | None = None,
    ) -> dict:
        """Create task with state 'submitted'. Auto-generate UUID v4 task_id and context_id."""

    async def transition(
        self,
        task_id: str,
        new_state: str,
        *,
        message: str | None = None,
        artifacts: list[dict] | None = None,
    ) -> dict:
        """Atomic state transition. Returns updated task."""

    async def get_task(self, task_id: str) -> dict | None: ...
    async def list_tasks(self, context_id: str | None, cursor: str | None, limit: int) -> dict: ...
    async def cancel_task(self, task_id: str) -> dict: ...
    async def subscribe(self, task_id: str) -> asyncio.Queue: ...
    async def unsubscribe(self, task_id: str, queue: asyncio.Queue) -> None: ...
```

**`transition()` steps:**
1. Get or create per-task `asyncio.Lock`.
2. Acquire lock.
3. `task = await store.get(task_id)` → raise `TaskNotFoundError` if None.
4. `current = task["status"]["state"]`.
5. If `new_state not in _TRANSITIONS[current]` → log ERROR, raise `InvalidStateTransitionError(current, new_state)`.
6. If `enable_history`: append current status to `task["history"]`.
7. Update `task["status"] = {"state": new_state, "message": message, "timestamp": utc_now()}`.
8. If `artifacts`: set `task["artifacts"] = artifacts`.
9. `await store.save(task)`.
10. Notify all subscribers: `queue.put_nowait(event_dict)`.
11. If new_state is terminal: garbage-collect lock and subscriber queues.

**`cancel_task(task_id)`:**
- Get task → if not found: raise `TaskNotFoundError` (maps to -32001).
- If terminal state: raise `TaskNotCancelableError` (maps to -32002).
- `await transition(task_id, "canceled")`.

**`subscribe(task_id)` / `unsubscribe(task_id, queue)`:**
- Creates a new `asyncio.Queue()`, adds to `_event_subscribers[task_id]`.
- Used by `StreamingHandler` and `PushNotificationManager` for reactive event delivery.

---

### 4. `TransportManager` — `server/transport.py`

Starlette route handlers for JSON-RPC dispatch and Agent Card serving.

```python
class TransportManager:
    def __init__(
        self,
        agent_card: dict,
        extended_card: dict | None,
        router: ExecutionRouter,
        task_manager: TaskManager,
        streaming_handler: StreamingHandler,
        push_manager: PushNotificationManager | None,
        *,
        cancel_on_disconnect: bool = True,
    ) -> None: ...

    async def handle_jsonrpc(self, request: Request) -> Response:
        """POST / — JSON-RPC 2.0 dispatch."""

    async def handle_agent_card(self, request: Request) -> JSONResponse:
        """GET /.well-known/agent.json"""

    async def handle_extended_card(self, request: Request) -> JSONResponse:
        """GET /agent/authenticatedExtendedCard"""

    async def handle_health(self, request: Request) -> JSONResponse:
        """GET /health"""

    async def handle_cancel(self, task_id: str, identity=None) -> dict:
        """Internal: cancel task, raise on non-cancelable."""
```

**`handle_jsonrpc` dispatch:**

| method | handler |
|---|---|
| `message/send` | `router.handle_message_send(params, identity)` |
| `message/stream` | `streaming_handler.handle_stream(params, identity)` → `StreamingResponse` |
| `tasks/get` | `task_manager.get_task(params["id"])` |
| `tasks/cancel` | `handle_cancel(params["id"])` |
| `tasks/list` | `task_manager.list_tasks(...)` |
| `tasks/resubscribe` | `streaming_handler.handle_resubscribe(params)` |
| `tasks/pushNotificationConfig/set` | `push_manager.set_config(params)` |
| `tasks/pushNotificationConfig/get` | `push_manager.get_config(params)` |
| `tasks/pushNotificationConfig/delete` | `push_manager.delete_config(params)` |
| unknown | `{"code": -32601, "message": "Method not found"}` |

**Validation:**
- `Content-Type != application/json` → HTTP 415.
- Body > 10 MB → HTTP 413.
- Unparseable JSON → JSON-RPC -32700.
- Missing `jsonrpc`, `method`, or `id` → JSON-RPC -32600.

**JSON-RPC response envelope:**
```python
def _ok_response(id, result):
    return {"jsonrpc": "2.0", "id": id, "result": result}

def _error_response(id, code, message, data=None):
    return {"jsonrpc": "2.0", "id": id, "error": {"code": code, "message": message, "data": data}}
```

**Agent Card response:**
- HTTP 200, `Content-Type: application/json`, `Cache-Control: max-age=300`.
- No auth required (exempt path).

**Extended card:**
- HTTP 404 if auth not configured.
- HTTP 401 if auth configured but no valid credentials.
- HTTP 200 with extended card JSON if authenticated.

**Health:**
```json
{"status": "ok", "module_count": 5, "uptime_seconds": 3600}
```

## File Structure

```
src/apcore_a2a/server/
    __init__.py         # exports: A2AServerFactory
    factory.py          # A2AServerFactory
    router.py           # ExecutionRouter
    task_manager.py     # TaskManager, InvalidStateTransitionError
    transport.py        # TransportManager
```

## Key Invariants

- `ExecutionRouter` uses duck-typing for both Registry and Executor
- `TaskManager` locks are per-task (not global) — concurrent tasks on different IDs proceed independently
- All state transitions validated against `_TRANSITIONS` table — no illegal transitions possible
- `TransportManager` routes to `StreamingHandler` for `message/stream` (separate component F-04)
- JSON-RPC -32001 used for ACL denials to mask resource existence (security)

## Test Modules

- `tests/server/test_server_factory.py`
- `tests/server/test_execution_router.py`
- `tests/server/test_task_manager.py`
- `tests/server/test_transport.py`
