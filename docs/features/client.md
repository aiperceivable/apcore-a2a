---
description: "Client module spec: A2AClient for calling remote A2A agents (send, stream, manage tasks) plus AgentCardFetcher for cached card discovery; depends only on httpx and stdlib."
---

# Feature: Client Module

| Field | Value |
|-------|-------|
| Feature ID | F-07 |
| Name | client |
| Priority | P0 |
| SRS Refs | FR-CLI-001, FR-CLI-002, FR-CLI-003, FR-CLI-004, FR-CLI-005 |
| Tech Design | §4.5 Client Module |
| Depends On | None (standalone — no server imports) |
| Blocks | F-08 (public-api re-exports A2AClient) |

## Purpose

HTTP client for calling remote A2A agents. Provides `A2AClient` for sending messages, streaming, and managing tasks. Provides `AgentCardFetcher` for cached Agent Card discovery. **Import constraint**: this module imports only `httpx` and the standard library — zero imports from `server/`, `auth/`, `storage/`, `starlette`, or `uvicorn`.

## Components

### 1. `A2AClient` — `client/client.py`

=== "Python"

    ```python
    class A2AClient:
        def __init__(
            self,
            url: str,
            *,
            auth: str | None = None,       # Bearer token e.g. "Bearer eyJ..."
            timeout: float = 30.0,          # HTTP request timeout seconds
            card_ttl: float = 300.0,        # Agent Card cache TTL seconds
        ) -> None:
            """Construct A2A client for a remote agent.

            Raises:
                ValueError: If url is not a valid HTTP/HTTPS URL.
            """

        @property
        async def agent_card(self) -> dict:
            """Fetch and cache the remote Agent Card (TTL-based)."""

        async def send_message(
            self,
            message: dict,
            *,
            metadata: dict | None = None,
            context_id: str | None = None,
        ) -> dict:
            """Send message/send JSON-RPC request. Returns Task dict.

            Raises:
                TaskNotFoundError: JSON-RPC error -32001.
                A2AServerError: JSON-RPC error -32603 (internal server error).
                A2AConnectionError: Network-level failure or HTTP error.
            """

        async def stream_message(
            self,
            message: dict,
            *,
            metadata: dict | None = None,
            context_id: str | None = None,
        ) -> AsyncGenerator[dict, None]:
            """Send message/stream and yield parsed SSE event dicts.

            Yields:
                TaskStatusUpdateEvent or TaskArtifactUpdateEvent dicts.
            Terminates when stream closes or a terminal TASK_STATE_* status is received.
            """

        async def get_task(self, task_id: str) -> dict:
            """Retrieve task state via tasks/get."""

        async def cancel_task(self, task_id: str) -> dict:
            """Cancel a task via tasks/cancel.

            Raises:
                TaskNotFoundError: -32001 if task not found.
                TaskNotCancelableError: -32002 if task is in terminal state.
            """

        async def list_tasks(
            self,
            context_id: str | None = None,
            limit: int = 50,
        ) -> dict:
            """List tasks via tasks/list.
            Returns {tasks: [...], nextCursor: str|None}.
            """

        async def discover(self) -> dict:
            """Convenience alias: fetch and return the Agent Card."""

        async def close(self) -> None:
            """Close the underlying HTTP client."""

        async def __aenter__(self) -> "A2AClient": ...
        async def __aexit__(self, *args) -> None: ...

        async def _jsonrpc_call(self, method: str, params: dict) -> dict:
            """POST JSON-RPC request. Returns result dict or raises typed error."""

        def _validate_url(self, url: str) -> None:
            """Validate url is well-formed HTTP/HTTPS. Raises ValueError."""
    ```

=== "TypeScript"

    ```typescript
    // src/client/client.ts — options are a single object; timeout is in milliseconds
    class A2AClient {
      // auth: Bearer token e.g. "Bearer eyJ..."; timeout default 30000 (ms);
      // cardTtl default 300 (seconds). Throws on an invalid HTTP/HTTPS url.
      constructor(url: string, opts?: { auth?: string; timeout?: number; cardTtl?: number })

      // Fetch and cache the remote Agent Card (TTL-based).
      get agentCard(): Promise<Record<string, unknown>>

      // Send message/send JSON-RPC request. Returns a Task dict.
      sendMessage(
        message: Record<string, unknown>,
        opts?: { metadata?: Record<string, unknown>; contextId?: string },
      ): Promise<Record<string, unknown>>

      // Send message/stream; yields parsed SSE event dicts until a terminal
      // TASK_STATE_* status is received or the stream closes.
      streamMessage(
        message: Record<string, unknown>,
        opts?: { metadata?: Record<string, unknown>; contextId?: string },
      ): AsyncGenerator<Record<string, unknown>>

      getTask(taskId: string): Promise<Record<string, unknown>>          // tasks/get
      cancelTask(taskId: string): Promise<Record<string, unknown>>       // tasks/cancel
      listTasks(opts?: { contextId?: string; limit?: number }): Promise<Record<string, unknown>>

      discover(): Promise<Record<string, unknown>>   // alias: fetch the Agent Card
      close(): void                                  // parity no-op
    }
    ```

=== "Rust"

    ```rust
    // src/client/client.rs — messages use serde_json::Value; timeouts are Duration.
    impl A2AClient {
        // new() panics on a bad URL; try_new() returns ClientResult<Self>.
        // auth is a Bearer token e.g. "Bearer eyJ..."; timeout default 30s; card_ttl default 300s.
        pub fn new(url: impl Into<String>) -> Self
        pub fn try_new(
            url: impl Into<String>,
            auth: Option<String>,
            timeout: Option<Duration>,
            card_ttl: Option<Duration>,
        ) -> ClientResult<Self>

        // Fetch and cache the remote Agent Card (TTL-based).
        pub async fn agent_card(&self) -> ClientResult<Value>

        // Send message/send JSON-RPC request. Returns a Task value.
        pub async fn send_message(
            &self,
            message: Value,
            metadata: Option<Value>,
            context_id: Option<String>,
        ) -> ClientResult<Value>

        // Send message/stream; the returned Stream yields events until a terminal
        // TASK_STATE_* status is received or the stream closes.
        pub fn stream_message(
            &self,
            message: Value,
            metadata: Option<Value>,
            context_id: Option<String>,
        ) -> impl Stream<Item = ClientResult<Value>> + '_

        pub async fn get_task(&self, task_id: &str) -> ClientResult<Value>     // tasks/get
        pub async fn cancel_task(&self, task_id: &str) -> ClientResult<Value>  // tasks/cancel
        pub async fn list_tasks(&self, context_id: Option<String>, limit: i64) -> ClientResult<Value>

        pub async fn discover(&self) -> ClientResult<Value>   // alias of agent_card
        pub async fn close(self)                              // parity no-op
    }
    ```

---

### 2. `AgentCardFetcher` — `client/card_fetcher.py`

=== "Python"

    ```python
    class AgentCardFetcher:
        def __init__(
            self,
            http: httpx.AsyncClient,
            base_url: str,
            *,
            ttl: float = 300.0,
        ) -> None:
            self._http = http
            # A2A 1.0 primary discovery path (the server also serves the 0.3
            # `/.well-known/agent.json` alias for backward compatibility).
            self._url = f"{base_url}/.well-known/agent-card.json"
            self._ttl = ttl
            self._cached: dict | None = None
            self._cached_at: float = 0.0

        async def fetch(self) -> dict:
            """Fetch Agent Card, returning cached version if within TTL.

            Steps:
            1. now = time.monotonic()
            2. If cached is not None and (now - cached_at) < ttl → return cached.
            3. GET self._url.
            4. If HTTP status != 200 → raise A2ADiscoveryError with message.
            5. Parse JSON → raise A2ADiscoveryError on parse error.
            6. Store in self._cached, update self._cached_at = now.
            7. Return cached dict.

            Raises:
                A2ADiscoveryError: HTTP error or invalid JSON in response.
            """
    ```

=== "TypeScript"

    ```typescript
    // src/client/card-fetcher.ts
    class AgentCardFetcher {
      // GETs `${baseUrl}/.well-known/agent-card.json` (A2A 1.0 primary discovery path;
      // the server also serves the 0.3 `/.well-known/agent.json` alias).
      // ttl in seconds (default 300); optional extra request headers.
      constructor(baseUrl: string, opts?: { ttl?: number; headers?: Record<string, string> })

      // Return the cached card if within TTL, otherwise fetch and cache it.
      // Throws A2ADiscoveryError on HTTP error or invalid JSON.
      fetch(): Promise<Record<string, unknown>>
    }
    ```

=== "Rust"

    ```rust
    // src/client/client.rs
    impl AgentCardFetcher {
        // GETs `${base_url}/.well-known/agent-card.json` (A2A 1.0 primary discovery path;
        // the server also serves the 0.3 `/.well-known/agent.json` alias).
        // DEFAULT_TTL_SECS = 300; with_ttl() overrides the cache TTL.
        pub fn new(http: reqwest::Client, base_url: impl Into<String>) -> Self
        pub fn with_ttl(http: reqwest::Client, base_url: impl Into<String>, ttl: Duration) -> Self

        // Return the cached card if within TTL, otherwise fetch and cache it.
        // Errors with A2AClientError::Discovery on HTTP error or invalid JSON.
        pub async fn fetch(&self) -> ClientResult<Value>
    }
    ```

---

### 3. Client Exceptions — `client/exceptions.py`

=== "Python"

    ```python
    class A2AClientError(Exception):
        """Base class for all client-side A2A errors."""

    class A2AConnectionError(A2AClientError):
        """Network-level failure: connection refused, timeout, DNS error."""

    class A2ADiscoveryError(A2AClientError):
        """Agent Card fetch failed: HTTP error or invalid JSON."""

    class TaskNotFoundError(A2AClientError):
        """JSON-RPC -32001: Task not found."""
        def __init__(self, task_id: str | None = None): ...

    class TaskNotCancelableError(A2AClientError):
        """JSON-RPC -32002: Task is in a terminal state."""
        def __init__(self, state: str | None = None): ...

    class A2AServerError(A2AClientError):
        """JSON-RPC -32603: Internal server error."""
        def __init__(self, message: str, code: int = -32603): ...
    ```

=== "TypeScript"

    ```typescript
    // src/client/exceptions.ts — Error subclasses
    class A2AClientError extends Error {}           // base class for all client errors
    class A2AConnectionError extends A2AClientError {}  // network-level failure
    class A2ADiscoveryError extends A2AClientError {}   // Agent Card fetch failed
    class TaskNotFoundError extends A2AClientError {}   // -32001; carries { taskId?: string }
    class TaskNotCancelableError extends A2AClientError {} // -32002; carries { state?: string }
    class A2AServerError extends A2AClientError {}      // -32603; carries { code: number }
    ```

=== "Rust"

    ```rust
    // src/client/client.rs — a single enum; pub type ClientResult<T> = Result<T, A2AClientError>;
    pub enum A2AClientError {
        Connection(String),                              // network-level failure
        Discovery(String),                               // Agent Card fetch failed
        TaskNotFound { task_id: Option<String> },        // JSON-RPC -32001
        TaskNotCancelable { state: Option<String> },     // JSON-RPC -32002
        Server { code: i32, message: String },           // JSON-RPC -32603 (or other codes)
        InvalidUrl(String),                              // malformed client URL
    }
    // A2AClientError::from_jsonrpc(code, message) maps a JSON-RPC error onto a variant.
    ```

---

## `send_message()` Detail

**Params construction:**
```python
params = {
    "message": message,  # {"role": "user", "parts": [...]}
    "metadata": metadata or {},
    "contextId": context_id,
}
# Remove None contextId:
if context_id is None:
    del params["contextId"]
```

**JSON-RPC request:**
```python
body = {
    "jsonrpc": "2.0",
    "id": str(uuid4()),
    "method": "message/send",
    "params": params,
}
response = await http.post(url, json=body)
```

**Response parsing:**
- HTTP non-2xx → raise `A2AConnectionError`.
- Parse JSON, check for `error` key → raise typed exception based on code.
- Return `result` dict.

**Error code → exception mapping:**

| JSON-RPC Code | Exception |
|---|---|
| -32001 | `TaskNotFoundError` |
| -32002 | `TaskNotCancelableError` |
| -32603 | `A2AServerError` |
| other | `A2AServerError(code=code)` |

---

## `stream_message()` Detail

**SSE parsing:**
```python
async with http.stream("POST", url, json=body) as response:
    async for line in response.aiter_lines():
        if line.startswith("data: "):
            data = json.loads(line[6:])
            yield data
            # A2A 1.0 has no `final` flag — stop on a terminal statusUpdate.
            su = data.get("statusUpdate") or {}
            if su.get("status", {}).get("state") in _TERMINAL_STATES:
                return
```

where `_TERMINAL_STATES = {"TASK_STATE_COMPLETED", "TASK_STATE_FAILED", "TASK_STATE_CANCELED", "TASK_STATE_REJECTED"}`.

Keepalive comment lines (`: keepalive`) are skipped silently.

---

## `_jsonrpc_call()` Detail

```python
async def _jsonrpc_call(self, method: str, params: dict) -> dict:
    body = {"jsonrpc": "2.0", "id": str(uuid4()), "method": method, "params": params}
    try:
        response = await self._http.post(f"{self._url}/", json=body)
        response.raise_for_status()
    except httpx.HTTPStatusError as e:
        raise A2AConnectionError(str(e)) from e
    except httpx.RequestError as e:
        raise A2AConnectionError(str(e)) from e
    data = response.json()
    if "error" in data:
        _raise_jsonrpc_error(data["error"])
    return data["result"]
```

---

## Constructor Validation

```python
def _validate_url(self, url: str) -> None:
    parsed = urlparse(url)
    if parsed.scheme not in ("http", "https") or not parsed.netloc:
        raise ValueError(f"Invalid A2A agent URL: {url!r} (must be http:// or https://)")
```

The validated URL is stored with trailing slash stripped: `self._url = url.rstrip("/")`.

The `httpx.AsyncClient` is initialized with:
- `timeout=timeout`
- `headers={"Authorization": auth}` if `auth` is not None; otherwise no Authorization header.

---

## Context Manager Usage

=== "Python"

    ```python
    async with A2AClient("https://agent.example.com", auth="Bearer eyJ...") as client:
        task = await client.send_message(
            {"role": "user", "parts": [{"text": "resize to 800x600"}]},
            metadata={"skillId": "image.resize"},
        )
    ```

=== "TypeScript"

    ```typescript
    import { A2AClient } from "apcore-a2a";

    const client = new A2AClient("https://agent.example.com", { auth: "Bearer eyJ..." });
    try {
      const task = await client.sendMessage(
        { role: "user", parts: [{ text: "resize to 800x600" }] },
        { metadata: { skillId: "image.resize" } },
      );
    } finally {
      client.close();
    }
    ```

=== "Rust"

    ```rust
    use apcore_a2a::A2AClient;
    use serde_json::json;

    // A2A 1.0 wire shape: role "ROLE_USER", text part { "text": ... }.
    let client = A2AClient::try_new(
        "https://agent.example.com",
        Some("Bearer eyJ...".into()),
        None,
        None,
    )?;
    let task = client.send_message(
        json!({ "role": "ROLE_USER", "parts": [{ "text": "resize to 800x600" }] }),
        Some(json!({ "skillId": "image.resize" })),
        None,
    ).await?;
    client.close().await;
    ```

Python's `__aexit__` calls `await self.close()` which calls `await self._http.aclose()`.
TypeScript's `close()` and Rust's `close()` are parity no-ops (no persistent connection to tear down).

---

## File Structure

```
src/apcore_a2a/client/
    __init__.py       # exports: A2AClient, AgentCardFetcher,
                      #          A2AClientError, A2AConnectionError, A2ADiscoveryError,
                      #          TaskNotFoundError, TaskNotCancelableError, A2AServerError
    client.py         # A2AClient
    card_fetcher.py   # AgentCardFetcher
    exceptions.py     # All client exceptions
```

## Key Invariants

- **Zero server imports**: `client/` contains no imports from `server/`, `auth/`, `storage/`, `starlette`, or `uvicorn`
- **Context manager safety**: `close()` always called on exit — no HTTP client leaks
- **Agent Card caching**: TTL-based, monotonic clock, no thread locks needed (async event loop)
- **Typed exceptions**: every JSON-RPC error maps to a specific typed exception — callers never need to inspect error codes
- **URL validation at construction time**: `ValueError` on invalid URL, not at call time
- **SSE stream termination**: stream ends when a terminal `TASK_STATE_*` status is received OR server closes connection (A2A 1.0 has no `final` flag)

## Test Module

`tests/client/test_a2a_client.py`
`tests/client/test_agent_card_fetcher.py`
