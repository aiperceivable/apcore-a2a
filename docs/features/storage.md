---
description: "Storage module spec: pluggable task persistence defining the TaskStore protocol (save/get/delete) with InMemoryTaskStore default; superseded by a2a-sdk's TaskStore."
---

# Feature: Storage Module

| Field | Value |
|-------|-------|
| Feature ID | F-02 |
| Name | storage |
| Priority | P0 |
| SRS Refs | FR-STR-001, FR-STR-002, FR-STR-003 |
| Tech Design | §4.7 Storage Module |
| Depends On | None |
| Blocks | F-03 (server-core, TaskManager needs store) |

> **Implementation Note**: The custom `TaskStore` protocol and `InMemoryTaskStore` described in this
> spec have been superseded by `a2a-sdk`'s `TaskStore` and `InMemoryTaskStore` (from
> `a2a.server.tasks`). The a2a-sdk interface exposes `save`, `get`, and `delete` only. Push
> notification config and context storage are handled internally by the SDK. This document reflects
> the original design intent.

## Purpose

Pluggable task persistence layer. Defines the `TaskStore` protocol and provides `InMemoryTaskStore` as the default implementation. Enables future Redis/PostgreSQL backends without code changes.

## Components

### 1. `TaskStore` Protocol — `storage/protocol.py`

=== "Python"

    ```python
    from typing import Protocol, runtime_checkable

    @runtime_checkable
    class TaskStore(Protocol):
        async def save(self, task: dict) -> None:
            """Persist task. Overwrites if task with same id exists."""
            ...

        async def get(self, task_id: str) -> dict | None:
            """Return task by ID, or None if not found."""
            ...

        async def list(
            self,
            context_id: str | None = None,
            cursor: str | None = None,
            limit: int = 50,
        ) -> dict:
            """List tasks. Returns {"tasks": [...], "nextCursor": str | None}.
            Filter by context_id if provided. Clamp limit to [1, 200].
            """
            ...

        async def delete(self, task_id: str) -> bool:
            """Delete task. Returns True if existed, False if not found."""
            ...

        async def save_push_config(self, task_id: str, config: dict) -> None:
            """Persist push notification config for a task."""
            ...

        async def get_push_config(self, task_id: str) -> dict | None:
            """Return push config for task, or None."""
            ...

        async def delete_push_config(self, task_id: str) -> bool:
            """Delete push config. Returns True if existed."""
            ...
    ```

=== "TypeScript"

    ```typescript
    // Re-exported from @a2a-js/sdk/server (see src/storage/index.ts).
    export interface TaskStore {
      save(taskId: string, task: TaskData, context?: unknown): Promise<void>;
      load(taskId: string, context?: unknown): Promise<TaskData | null>;
      delete(taskId: string, context?: unknown): Promise<void>;
      list(...args: unknown[]): Promise<TaskData[]>;
    }
    ```

=== "Rust"

    ```rust
    use async_trait::async_trait;
    use serde_json::Value;
    use apcore_a2a::{CallContext, ListParams, StoreError};

    #[async_trait]
    pub trait TaskStore: Send + Sync {
        async fn save(&self, task_id: &str, task: Value, ctx: &CallContext)
            -> Result<(), StoreError>;
        async fn get(&self, task_id: &str, ctx: &CallContext)
            -> Result<Option<Value>, StoreError>;
        async fn delete(&self, task_id: &str, ctx: &CallContext)
            -> Result<(), StoreError>;
        async fn list(&self, params: &ListParams, ctx: &CallContext)
            -> Result<Vec<Value>, StoreError>;
    }

    // Push notification configs live in a matching, separately pluggable trait,
    // so a persistent deployment restores tasks and their webhook targets
    // together.
    #[async_trait]
    pub trait PushConfigStore: Send + Sync {
        async fn save(&self, task_id: &str, config: Value, ctx: &CallContext)
            -> Result<(), StoreError>;
        async fn get(&self, task_id: &str, ctx: &CallContext)
            -> Result<Option<Value>, StoreError>;
        async fn delete(&self, task_id: &str, ctx: &CallContext)
            -> Result<(), StoreError>;
    }
    ```

    **Owner scoping is part of the contract.** Every method must be scoped by
    `ctx.owner()`: `get` and `delete` on another owner's task must behave
    exactly as they do for a task that does not exist (never a distinguishable
    error, or task ids become probeable), and `list` must return only the
    caller's tasks. Bucketing on `(owner, task_id)` satisfies all of it at
    once. A store that ignores `ctx` disables task isolation entirely; no
    language in this project can enforce it — upstream `a2a-python` states the
    same requirement as a SHOULD.

**Runtime checkable (Python only)**: `isinstance(store, TaskStore)` must work for duck-typed custom
implementations. TypeScript checks structurally at compile time; Rust requires an explicit `impl
TaskStore`.

---

### 2. `InMemoryTaskStore` — `storage/memory.py`

Default implementation using `dict` + `asyncio.Lock`.

=== "Python"

    ```python
    class InMemoryTaskStore:
        def __init__(self, *, max_tasks: int = 10_000) -> None:
            self._tasks: dict[str, dict] = {}
            self._push_configs: dict[str, dict] = {}
            self._lock = asyncio.Lock()
            self._max_tasks = max_tasks
    ```

=== "TypeScript"

    ```typescript
    import { InMemoryTaskStore } from "apcore-a2a";

    // Re-exported from @a2a-js/sdk/server; the SDK manages its own
    // internal task map. Construct with no arguments.
    const taskStore = new InMemoryTaskStore();
    ```

=== "Rust"

    ```rust
    use apcore_a2a::{InMemoryPushConfigStore, InMemoryTaskStore};

    // Backed by a Mutex<HashMap<OwnerId, HashMap<String, Value>>> — bucketed by
    // owner, so isolation is a property of the data structure. Both construct
    // with no arguments.
    let task_store = InMemoryTaskStore::new();
    let push_config_store = InMemoryPushConfigStore::new();
    ```

**Behavior:**

| Method | Implementation |
|---|---|
| `save(task)` | Acquire lock → if len >= max_tasks and task is new → raise OverflowError → `_tasks[task["id"]] = task` → release |
| `get(task_id)` | Return `_tasks.get(task_id)` (no lock needed for reads with GIL) |
| `list(context_id, cursor, limit)` | Filter by context_id, apply cursor-based pagination, clamp limit to [1, 200] |
| `delete(task_id)` | Acquire lock → pop and return True if existed |
| `save_push_config` | Acquire lock → store in `_push_configs[task_id]` |
| `get_push_config` | Return `_push_configs.get(task_id)` |
| `delete_push_config` | Acquire lock → pop from `_push_configs` |

**Concurrency**: Single `asyncio.Lock` guards all write operations. Read operations (`get`) are safe without locking in CPython (GIL-protected dict read). This is correct for an async event loop — operations are non-blocking.

**Pagination**: Cursor is the last task ID returned. Next page starts after the cursor. Tasks ordered by insertion order (Python dict ordering).

**Capacity**: `max_tasks=10_000` default. Raises `OverflowError("Task store capacity exceeded")` when at capacity and a new task is saved.

---

## Data Model — Task Dict

All tasks stored as plain Python dicts (JSON-serializable):

```python
{
    "id": "uuid-v4-string",                    # required, unique
    "contextId": "uuid-v4-string",             # required
    "status": {
        "state": "TASK_STATE_SUBMITTED",       # current state (A2A 1.0 TaskState enum)
        "message": dict | None,                # optional status Message
        "timestamp": "2026-03-03T10:00:00Z",   # ISO 8601 UTC
    },
    "artifacts": [],                           # list of Artifact dicts
    "history": [],                             # list of past TaskStatus dicts (if enabled)
    "metadata": {},                            # client-provided metadata
    "kind": "task",                            # A2A discriminator
}
```

**PushNotificationConfig dict:**
```python
{
    "taskId": "uuid-v4-string",
    "url": "https://...",       # webhook URL
    "token": str | None,        # opaque validation token
    "authentication": dict | None,
    "status": "active" | "failed",
}
```

## Actual Implementation

The storage module delegates entirely to the `a2a-sdk` library. The `storage/__init__.py` re-exports
`TaskStore` and `InMemoryTaskStore` from `a2a.server.tasks`:

=== "Python"

    ```python
    # src/apcore_a2a/storage/__init__.py
    from a2a.server.tasks import TaskStore, InMemoryTaskStore

    __all__ = ["TaskStore", "InMemoryTaskStore"]
    ```

=== "TypeScript"

    ```typescript
    // src/storage/index.ts
    export { InMemoryTaskStore } from "@a2a-js/sdk/server";
    export type { TaskStore } from "@a2a-js/sdk/server";
    ```

=== "Rust"

    ```rust
    // src/storage/mod.rs — defined in this crate, not re-exported from an SDK:
    //   TaskStore, PushConfigStore, InMemoryTaskStore, InMemoryPushConfigStore,
    //   CallContext, OwnerId, ListParams, StoreError
    // All are re-exported at the crate root (src/lib.rs).
    ```

The a2a-sdk 1.0 `TaskStore` interface exposes `save`, `get`, `delete` and `list`, each taking a
`ServerCallContext`; its `InMemoryTaskStore` and `DatabaseTaskStore` bucket by an owner resolved from
that context via an injectable `OwnerResolver`. Push notification config is handled by the SDK's own
`PushNotificationConfigStore`. The custom `protocol.py`, `memory.py`, cursor pagination, and
`max_tasks` capacity limiting described above are **not implemented** — they reflect the original
design intent before the a2a-sdk provided these components.

Rust is the exception: it defines its own `TaskStore` (there is no Rust a2a-sdk), so the
owner-carrying `CallContext` and the separate `PushConfigStore` are implemented in this project. The
resulting contract is deliberately the same one upstream states.

## File Structure

```
src/apcore_a2a/storage/
    __init__.py     # re-exports: TaskStore, InMemoryTaskStore from a2a-sdk
```

## Key Invariants

- `TaskStore` protocol methods: `save`, `get`, `delete` (provided by a2a-sdk)
- `InMemoryTaskStore` is the default when no task_store is provided to `serve()`
- Push notification config is managed internally by a2a-sdk's `DefaultRequestHandler`

## Test Module

Storage is tested indirectly through server integration tests.
