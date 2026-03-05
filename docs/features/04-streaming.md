# Feature: Streaming Handler

| Field | Value |
|-------|-------|
| Feature ID | F-04 |
| Name | streaming |
| Priority | P0 |
| SRS Refs | FR-MSG-002, FR-MSG-005, FR-MSG-006, FR-TSK-007 |
| Tech Design | §4.4.5 StreamingHandler |
| Depends On | F-03 (server-core — TaskManager, ExecutionRouter) |
| Blocks | F-08 (public-api) |

> **Implementation Note**: The `StreamingHandler` class described in this spec has been superseded by
> `a2a-sdk`'s `DefaultRequestHandler` + `InMemoryQueueManager`. SSE streaming behavior is otherwise
> equivalent. This document reflects the original design intent.

## Purpose

> **Note**: `cancel_on_disconnect` is accepted for backward compatibility but has no effect in the current implementation (a2a-sdk's `DefaultRequestHandler` does not support disabling cancel-on-disconnect). The parameter will be removed in a future version.

Implements A2A SSE streaming for `message/stream` and `tasks/resubscribe`. Converts task state transitions and streaming executor output into typed SSE events delivered to the client.

## Component: `StreamingHandler` — `server/streaming.py`

```python
class StreamingHandler:
    def __init__(
        self,
        task_manager: TaskManager,
        router: ExecutionRouter,
        *,
        cancel_on_disconnect: bool = True,
    ) -> None:
        self._task_manager = task_manager
        self._router = router
        self._cancel_on_disconnect = cancel_on_disconnect
        self._event_counter: dict[str, int] = {}  # task_id -> monotonic event id

    async def handle_stream(
        self,
        params: dict,
        identity: object | None = None,
    ) -> StreamingResponse:
        """Create SSE response for message/stream JSON-RPC request."""

    async def handle_resubscribe(
        self,
        params: dict,
    ) -> StreamingResponse | dict:
        """Reconnect to existing task's SSE stream (tasks/resubscribe)."""

    def _format_sse_event(self, event: dict, event_id: int) -> str:
        """Format as SSE: 'id: {n}\\ndata: {json}\\n\\n'"""

    def _make_status_event(self, task_id: str, state: str, message: str | None = None, artifacts: list | None = None) -> dict:
        """Build TaskStatusUpdateEvent dict."""

    def _make_artifact_event(self, task_id: str, parts: list[dict], append: bool, last_chunk: bool) -> dict:
        """Build TaskArtifactUpdateEvent dict."""
```

## `handle_stream()` Logic

**Precondition**: params contains a valid message with `metadata.skillId`.

**Steps:**

1. Validate `skillId` present in params — return JSON-RPC error dict (not SSE) if missing.
2. Create task via `task_manager.create_task(context_id=...)` → `task_id`.
3. Subscribe to task events: `queue = await task_manager.subscribe(task_id)`.
4. Return `StreamingResponse(generator, media_type="text/event-stream", headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"})`.

**Generator (`_stream_generator`):**

```python
async def _stream_generator(task_id, queue, params, identity):
    # 1. Emit submitted event immediately (< 50ms from request)
    event_id = _next_id(task_id)
    yield _format_sse_event(_make_status_event(task_id, "submitted"), event_id)

    # 2. Start execution in background
    bg_task = asyncio.create_task(
        router.handle_message_stream(params, task_id, identity)
    )

    # 3. Forward events from queue
    try:
        while True:
            event = await asyncio.wait_for(queue.get(), timeout=30.0)
            event_id = _next_id(task_id)
            yield _format_sse_event(event, event_id)
            if event.get("final"):
                break
    except asyncio.TimeoutError:
        # Send keepalive comment
        yield ": keepalive\n\n"
    except GeneratorExit:
        # Client disconnected
        if cancel_on_disconnect:
            await task_manager.cancel_task(task_id)
    finally:
        await task_manager.unsubscribe(task_id, queue)
        bg_task.cancel()
```

## SSE Event Format

All events follow the SSE spec (`text/event-stream`):

```
id: 1
data: {"type":"TaskStatusUpdateEvent","taskId":"abc-123","contextId":"ctx-xyz","status":{"state":"submitted","timestamp":"2026-03-03T10:00:00.000Z"},"final":false}

id: 2
data: {"type":"TaskStatusUpdateEvent","taskId":"abc-123","contextId":"ctx-xyz","status":{"state":"working","timestamp":"2026-03-03T10:00:00.050Z"},"final":false}

id: 3
data: {"type":"TaskArtifactUpdateEvent","taskId":"abc-123","contextId":"ctx-xyz","artifact":{"artifactId":"art-001","parts":[{"kind":"data","data":{"progress":50},"mediaType":"application/json"}]},"append":false,"lastChunk":false,"final":false}

id: 4
data: {"type":"TaskStatusUpdateEvent","taskId":"abc-123","contextId":"ctx-xyz","status":{"state":"completed","timestamp":"2026-03-03T10:00:05.000Z"},"artifacts":[...],"final":true}

```

**`final: true`** signals the client to close the SSE connection.

## `handle_resubscribe()` Logic

1. Extract `task_id = params.get("id")` → error -32001 if missing.
2. `task = await task_manager.get_task(task_id)` → error -32001 if None.
3. If task state is terminal (`completed`, `failed`, `canceled`):
   - Return `StreamingResponse` with single event: current `TaskStatusUpdateEvent` with `final: true`.
4. If task state is active:
   - Subscribe to events, start SSE stream from current state.
   - First event: current `TaskStatusUpdateEvent` for current state.
   - Continue streaming until terminal event.

## Artifact Streaming

When the executor supports streaming (`executor.stream()` exists), the `ExecutionRouter` yields chunk dicts. `StreamingHandler` converts each chunk to a `TaskArtifactUpdateEvent`:

```python
# First chunk: append=False, lastChunk=False
# Intermediate chunks: append=True, lastChunk=False
# Final chunk: append=True, lastChunk=True
```

For non-streaming executors: single artifact emitted with the completed status event.

## Keepalive

Every 30 seconds without a task event, emit SSE comment line `": keepalive\n\n"` to prevent proxy/load balancer connection termination.

## File Structure

```
src/apcore_a2a/server/
    streaming.py    # StreamingHandler
```

## Key Invariants

- First SSE event (submitted) emitted within 50ms of request receipt
- `final: true` always on the last event
- Client disconnect with `cancel_on_disconnect=True` triggers task cancellation
- Keepalive every 30s for long-running tasks
- `event_id` is monotonically increasing per-task

## Test Module

`tests/server/test_streaming_handler.py`
