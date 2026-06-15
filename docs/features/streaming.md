---
description: "Streaming module spec: A2A SSE streaming for message/stream and tasks/resubscribe, converting task transitions and executor output into typed SSE events; superseded by a2a-sdk."
---

# Feature: Streaming Handler

| Field | Value |
|-------|-------|
| Feature ID | F-04 |
| Name | streaming |
| Priority | P0 |
| SRS Refs | FR-MSG-002, FR-MSG-005, FR-MSG-006 |
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
            # A2A 1.0 removed the `final` flag. The stream ends when a
            # statusUpdate carries a terminal TaskState
            # (TASK_STATE_COMPLETED / TASK_STATE_FAILED / TASK_STATE_CANCELED /
            # TASK_STATE_REJECTED).
            su = event.get("statusUpdate")
            if su and su.get("status", {}).get("state") in _TERMINAL_STATES:
                break
    except asyncio.TimeoutError:
        # Send keepalive comment
        yield ": keepalive\n\n"
    except GeneratorExit:
        # Client disconnected. `cancel_on_disconnect` is a deprecated no-op
        # (a2a-sdk's DefaultRequestHandler does not support disabling
        # disconnect-driven cancellation), so no explicit cancel is issued here.
        pass
    finally:
        await task_manager.unsubscribe(task_id, queue)
        bg_task.cancel()
```

## SSE Event Format

All events follow the SSE spec (`text/event-stream`). Each `data:` line carries
one **A2A 1.0** event. A2A 1.0 is protobuf-derived: events are a `oneof`
discriminated by the wrapper key (`task` / `statusUpdate` / `artifactUpdate` /
`msg`) — there is no `type`/`kind` field and no `final` flag. Field names are
camelCase, and `TaskState` serializes as its full enum name
(`TASK_STATE_SUBMITTED`, `TASK_STATE_WORKING`, `TASK_STATE_COMPLETED`, …).
`Part` is a flattened `oneof`: a text part is `{"text":"…"}`, a data part is
`{"data":{…}}` (no `type` discriminator).

```
id: 1
data: {"statusUpdate":{"taskId":"abc-123","contextId":"ctx-xyz","status":{"state":"TASK_STATE_SUBMITTED","timestamp":"2026-03-03T10:00:00.000Z"}}}

id: 2
data: {"statusUpdate":{"taskId":"abc-123","contextId":"ctx-xyz","status":{"state":"TASK_STATE_WORKING","timestamp":"2026-03-03T10:00:00.050Z"}}}

id: 3
data: {"artifactUpdate":{"taskId":"abc-123","contextId":"ctx-xyz","artifact":{"artifactId":"art-001","parts":[{"data":{"progress":50}}]},"append":false,"lastChunk":false}}

id: 4
data: {"statusUpdate":{"taskId":"abc-123","contextId":"ctx-xyz","status":{"state":"TASK_STATE_COMPLETED","timestamp":"2026-03-03T10:00:05.000Z"}}}

```

A **terminal `statusUpdate`** (`TASK_STATE_COMPLETED`, `TASK_STATE_FAILED`,
`TASK_STATE_CANCELED`, or `TASK_STATE_REJECTED`) signals the client to close the
SSE connection. (In A2A 0.3 this was the `final: true` flag, which 1.0 removed.)

## `handle_resubscribe()` Logic

1. Extract `task_id = params.get("id")` → error -32001 if missing.
2. `task = await task_manager.get_task(task_id)` → error -32001 if None.
3. If task state is terminal (`TASK_STATE_COMPLETED`, `TASK_STATE_FAILED`, `TASK_STATE_CANCELED`, `TASK_STATE_REJECTED`):
   - Return `StreamingResponse` with a single `statusUpdate` event carrying the current terminal state.
4. If task state is active:
   - Subscribe to events, start SSE stream from current state.
   - First event: current `statusUpdate` for current state.
   - Continue streaming until a terminal `statusUpdate` is emitted.

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

- First SSE event (`statusUpdate` with `TASK_STATE_SUBMITTED`) emitted within 50ms of request receipt
- The last event is always a terminal `statusUpdate` (`TASK_STATE_COMPLETED` / `TASK_STATE_FAILED` / `TASK_STATE_CANCELED` / `TASK_STATE_REJECTED`); A2A 1.0 has no `final` flag
- `cancel_on_disconnect` is a deprecated no-op (a2a-sdk's `DefaultRequestHandler` does not support disabling disconnect-driven cancellation); disconnect-cancel behavior is governed by the SDK
- Long-running streams are bounded by apcore's `global_deadline` (mapped from `execution_timeout`); on expiry the stream ends with a failed `statusUpdate`
- Keepalive every 30s for long-running tasks
- `event_id` is monotonically increasing per-task

## Test Module

`tests/server/test_streaming_handler.py`
