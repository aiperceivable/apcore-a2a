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

**Runtime checkable**: `isinstance(store, TaskStore)` must work for duck-typed custom implementations.

---

### 2. `InMemoryTaskStore` — `storage/memory.py`

Default implementation using `dict` + `asyncio.Lock`.

```python
class InMemoryTaskStore:
    def __init__(self, *, max_tasks: int = 10_000) -> None:
        self._tasks: dict[str, dict] = {}
        self._push_configs: dict[str, dict] = {}
        self._lock = asyncio.Lock()
        self._max_tasks = max_tasks
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
        "state": "submitted",                  # current state
        "message": str | None,                 # optional status message
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

## File Structure

```
src/apcore_a2a/storage/
    __init__.py     # exports: TaskStore, InMemoryTaskStore
    protocol.py     # TaskStore Protocol
    memory.py       # InMemoryTaskStore
```

## Key Invariants

- `TaskStore` is a `runtime_checkable` Protocol — custom implementations checked at startup
- `InMemoryTaskStore` is thread-safe via `asyncio.Lock`
- `save()` is idempotent for existing tasks (overwrites by ID)
- `list()` result order is stable (insertion order)
- `max_tasks` prevents memory exhaustion

## Test Module

`tests/storage/test_in_memory_task_store.py`
