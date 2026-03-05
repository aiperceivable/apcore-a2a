# Feature: Explorer UI

| Field | Value |
|-------|-------|
| Feature ID | F-10 |
| Name | explorer |
| Priority | P2 |
| SRS Refs | FR-EXP-001 |
| Tech Design | §4.8 Explorer Module |
| Depends On | F-03 (server-core — ExecutionRouter, agent_card) |
| Blocks | None |

## Purpose

Browser-based interactive UI for exploring and testing an A2A agent. Mounted as a Starlette sub-application at a configurable URL prefix (default: `/explorer`). Delivered as a single self-contained HTML file — no external CDN dependencies.

## Component: `create_explorer_mount()` — `explorer/__init__.py`

```python
def create_explorer_mount(
    agent_card: dict,
    router: object,  # duck-typed: DefaultRequestHandler or ExecutionRouter
    *,
    explorer_prefix: str = "/explorer",
    authenticator: object | None = None,
) -> Mount:
    """Create a Starlette Mount for the Explorer UI.

    Args:
        agent_card: Pre-computed Agent Card dict (served to Explorer JS).
        router: Request handler for proxying test requests (duck-typed).
        explorer_prefix: URL prefix at which the Explorer is mounted.
        authenticator: Optional Authenticator for Explorer POST routes.

    Returns:
        A Starlette Mount that can be added to the parent app's routes.
    """
```

**Mount routes:**

| Method | Path | Handler | Auth Required |
|--------|------|---------|---------------|
| `GET` | `{explorer_prefix}/` | Serve `index.html` | No |
| `GET` | `{explorer_prefix}/agent-card` | Return agent card JSON | No |

Explorer GET endpoints are **exempt** from JWT authentication even when `auth` is configured. This is enforced by adding `explorer_prefix` to `AuthMiddleware.exempt_prefixes`.

---

## Explorer UI Features (`index.html`)

The single-file UI provides:

### Agent Card Panel
- Agent name, description, version, URL
- Capabilities: streaming, push notifications, state history
- Security schemes listing

### Skills Browser
- List all skills with: ID, name, description, tags
- Expand each skill to view:
  - Input/output MIME modes
  - Examples (up to 10)
  - apcore annotations (`readonly`, `destructive`, `idempotent`, `requires_approval`)
  - Input schema (formatted JSON)

### Message Composer
- Skill selector dropdown (populated from agent card)
- Message editor (text input or JSON DataPart)
- Context ID field (optional, pre-filled with generated UUID)
- **Send** button → `message/send` JSON-RPC call → display Task result
- **Stream** button → `message/stream` → live SSE event log

### SSE Stream Viewer
- Live-rendered event list
- Event type badges (status / artifact)
- State transitions highlighted
- Artifact parts displayed inline (text rendered, JSON pretty-printed)

### Task State Viewer
- Input task ID → fetch via `tasks/get`
- Display current state, timestamp, history timeline
- Cancel button → `tasks/cancel`

---

## Implementation Notes

### Single HTML File
```
src/apcore_a2a/explorer/index.html
```

- No external CDN (fonts, JS libs) — fully self-contained.
- Agent Card fetched from `GET {explorer_prefix}/agent-card` on page load.
- JSON-RPC calls made via `fetch()` to the parent server's `POST /`.
- SSE stream via `EventSource` API pointed at parent server's `POST /`.
- Auth token (if configured) stored in sessionStorage, sent as `Authorization: Bearer` header.

### Starlette Mount
```python
from starlette.routing import Mount, Route
from starlette.responses import HTMLResponse
from starlette.staticfiles import StaticFiles

def create_explorer_mount(agent_card, router, *, explorer_prefix="/explorer", authenticator=None):
    html_path = Path(__file__).parent / "index.html"

    async def serve_index(request):
        return HTMLResponse(html_path.read_text())

    async def serve_agent_card(request):
        return JSONResponse(agent_card)

    return Mount(explorer_prefix, routes=[
        Route("/", endpoint=serve_index),
        Route("/agent-card", endpoint=serve_agent_card),
    ])
```

---

## Auth Exemption

When `A2AServerFactory` mounts the Explorer, it passes `explorer_prefix` to `AuthMiddleware`:

```python
# In A2AServerFactory.create():
if explorer:
    explorer_mount = create_explorer_mount(agent_card, router, explorer_prefix=explorer_prefix)
    routes.append(explorer_mount)
    exempt_prefixes.add(explorer_prefix)
```

This means any `GET {explorer_prefix}/*` request bypasses JWT validation.

---

## File Structure

```
src/apcore_a2a/explorer/
    __init__.py    # create_explorer_mount()
    index.html     # Self-contained Explorer UI
```

## Key Invariants

- Explorer is mounted only when `explorer=True` in `serve()` / `A2AServerFactory.create()`
- All Explorer routes are exempt from JWT authentication (public dev tool)
- `index.html` is self-contained — no external network requests from the UI
- Agent Card data served at `{explorer_prefix}/agent-card` (not re-fetching `/.well-known/agent.json` to avoid CORS)
- `create_explorer_mount()` has no dependency on auth or storage modules

## Test Module

`tests/explorer/test_explorer.py`
