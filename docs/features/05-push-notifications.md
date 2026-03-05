# Feature: Push Notification Manager

| Field | Value |
|-------|-------|
| Feature ID | F-05 |
| Name | push-notifications |
| Priority | P1 |
| SRS Refs | FR-PSH-001, FR-PSH-002, FR-PSH-003, FR-PSH-004 |
| Tech Design | §4.4.6 PushNotificationManager |
| Depends On | F-02 (storage), F-03 (server-core — TaskManager) |
| Blocks | F-08 (public-api, when push_notifications=True) |

> **Implementation Note**: The `PushNotificationManager` described in this spec has been superseded
> by `a2a-sdk`'s `InMemoryPushNotificationConfigStore` (from
> `a2a.server.tasks.inmemory_push_notification_config_store`). Push notification delivery and retry
> logic are handled internally by the SDK. This document reflects the original design intent.

## Purpose

Webhook-based async delivery of task state changes. Clients register a webhook URL for a task; the server delivers `TaskStatusUpdateEvent` payloads via HTTP POST with retry on failure.

## Component: `PushNotificationManager` — `server/push.py`

```python
class PushNotificationManager:
    _MAX_RETRIES = 3
    _RETRY_DELAYS = [1.0, 2.0, 4.0]  # seconds, exponential backoff

    def __init__(self, store: TaskStore) -> None:
        self._store = store
        self._http_client = httpx.AsyncClient(timeout=10.0)

    # JSON-RPC handlers
    async def set_config(self, params: dict) -> dict: ...
    async def get_config(self, params: dict) -> dict: ...
    async def delete_config(self, params: dict) -> dict: ...

    # Called by TaskManager on state transitions
    async def notify(self, task_id: str, event: dict) -> None: ...

    async def close(self) -> None:
        """Close HTTP client."""
```

## `set_config()` — Register Webhook

**Input params:**
```json
{
  "id": "task-uuid",
  "pushNotificationConfig": {
    "url": "https://webhook.example.com/notify",
    "token": "opaque-validation-token",
    "authentication": {"schemes": ["Bearer"], "credentials": "..."}
  }
}
```

**Validation (in order):**
1. `params.id` missing → error -32602 `"Missing required parameter: id"`.
2. `params.pushNotificationConfig.url` missing → error -32602 `"Missing required parameter: pushNotificationConfig.url"`.
3. URL not parseable as valid HTTP/HTTPS → error -32602 `"Invalid URL format"`.
4. URL scheme is not `https://` (in non-test mode) → error -32602 `"Webhook URL must use HTTPS"`.
5. URL resolves to loopback (127.x, ::1, localhost) or RFC-1918 private range → error -32602 `"Webhook URL cannot target private/loopback addresses"`.
6. Task not found in store → error -32001 `"Task not found"`.
7. Save config via `store.save_push_config(task_id, config)`.
8. Return stored config dict.

## `get_config()` — Retrieve Webhook Config

**Input params:** `{"id": "task-uuid"}`

**Steps:**
1. Task not found → error -32001.
2. Config not found → error -32001 `"Push notification config not found"`.
3. Return config dict.

## `delete_config()` — Remove Webhook Config

**Input params:** `{"id": "task-uuid"}`

**Steps:**
1. Task not found → error -32001.
2. Config not found → error -32001.
3. `await store.delete_push_config(task_id)`.
4. Return `{}`.

## `notify()` — Deliver Event via Webhook

Called by `TaskManager` after each state transition as a fire-and-forget background task:

```python
asyncio.create_task(push_manager.notify(task_id, event))
```

**Steps:**
1. `config = await store.get_push_config(task_id)` → silently return if None.
2. If `config["status"] == "failed"` → skip (exhausted retries previously).
3. Build POST request:
   - URL: `config["url"]`
   - Body: JSON-serialized event dict
   - Headers:
     - `Content-Type: application/json`
     - `X-A2A-Notification-Token: {config["token"]}` (if token present)
     - `Authorization: {config["authentication"]["credentials"]}` (if authentication present)
4. Send POST, check response:
   - HTTP 2xx → success, return.
   - Other → retry.
5. **Retry logic** (up to 3 attempts):
   ```python
   for attempt, delay in enumerate(self._RETRY_DELAYS):
       await asyncio.sleep(delay)
       # retry POST
       if response.status_code // 100 == 2:
           return
   # All retries exhausted:
   config["status"] = "failed"
   await store.save_push_config(task_id, config)
   logger.error("Push notification delivery failed after 3 retries for task %s", task_id)
   ```
6. Log each retry attempt at WARNING level: `"Push notification retry {attempt+1}/3 for task {task_id}"`.

## Delivery Payload Format

POST body is the same event dict emitted by SSE:
```json
{
  "type": "TaskStatusUpdateEvent",
  "taskId": "abc-123",
  "contextId": "ctx-xyz",
  "status": {"state": "completed", "timestamp": "2026-03-03T10:00:05.000Z"},
  "artifacts": [...],
  "final": true
}
```

## Integration with TaskManager

`PushNotificationManager.notify()` is registered as a callback in `TaskManager`:

```python
# In TaskManager.transition():
if self._push_manager:
    asyncio.create_task(self._push_manager.notify(task_id, event))
```

The `PushNotificationManager` is optional — `TaskManager` works without it.

## Security

- SSRF protection: reject loopback and RFC-1918 URLs in production mode.
- Test mode (configurable): allow localhost URLs for testing.
- Token delivered as `X-A2A-Notification-Token` header for client-side validation.
- Webhook URL must be HTTPS in production.

## File Structure

```
src/apcore_a2a/server/
    push.py     # PushNotificationManager
```

## Key Invariants

- `notify()` is always fire-and-forget — never awaited by the caller
- Max 3 retries with delays [1s, 2s, 4s]
- Failed configs marked `status: "failed"` — no further delivery attempts
- Token header always sent if config has a token
- SSRF protection enforced at `set_config()` time, not delivery time

## Test Module

`tests/server/test_push_notification_manager.py`
