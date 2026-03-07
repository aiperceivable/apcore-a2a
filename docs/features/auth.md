# Feature: Auth Module

| Field | Value |
|-------|-------|
| Feature ID | F-06 |
| Name | auth |
| Priority | P1 |
| SRS Refs | FR-AUT-001, FR-AUT-002, FR-AUT-003, FR-AUT-004 |
| Tech Design | §4.6 Auth Module |
| Depends On | None (standalone) |
| Blocks | F-03 (server-core uses AuthMiddleware), F-08 (public-api exposes auth param) |

## Purpose

JWT/Bearer authentication for the A2A server. Validates tokens, maps claims to apcore `Identity`, bridges identity to the Executor via ASGI ContextVar. Mirrors apcore-mcp's auth module pattern.

## Components

### 1. `Authenticator` Protocol — `auth/protocol.py`

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Authenticator(Protocol):
    def authenticate(self, headers: dict[str, str]) -> object | None:
        """Validate credentials from HTTP headers.

        Args:
            headers: Lowercase-keyed HTTP headers dict
                     (e.g., {"authorization": "Bearer eyJ..."}).

        Returns:
            Identity object on success, None on invalid/missing credentials.
            Must never raise — return None for any auth failure.
        """
        ...

    def security_schemes(self) -> dict:
        """Return A2A-compatible security scheme dict for Agent Card declaration.

        Returns a dict keyed by scheme name, e.g.:
            {"bearerAuth": {"type": "http", "scheme": "bearer", "bearerFormat": "JWT"}}
        """
        ...
```

**Runtime checkable**: `isinstance(auth, Authenticator)` returns True for any duck-typed implementation.

---

### 2. `ClaimMapping` — `auth/jwt.py`

```python
from dataclasses import dataclass, field

@dataclass
class ClaimMapping:
    id_claim: str = "sub"          # JWT claim → Identity.id
    type_claim: str = "type"       # JWT claim → Identity.type
    roles_claim: str = "roles"     # JWT claim → Identity.roles (list[str])
    attrs_claims: list[str] = field(default_factory=list)  # extra claims → Identity.attrs
```

---

### 3. `JWTAuthenticator` — `auth/jwt.py`

```python
class JWTAuthenticator:
    def __init__(
        self,
        key: str,
        *,
        algorithms: list[str] | None = None,   # Default: ["HS256"]
        audience: str | None = None,
        issuer: str | None = None,
        claim_mapping: ClaimMapping | None = None,
        require_claims: list[str] | None = None,  # Default: ["sub"]
    ) -> None: ...

    def authenticate(self, headers: dict[str, str]) -> object | None:
        """Validate Bearer JWT token.

        Steps:
        1. Extract Authorization header → must be "Bearer {token}".
           Missing or non-Bearer → return None.
        2. Decode JWT with PyJWT:
           - Verify signature with self._key.
           - Verify expiration (exp claim).
           - Verify issuer (iss) if configured.
           - Verify audience (aud) if configured.
           - Any PyJWT.InvalidTokenError → return None (never raise).
        3. Check required_claims all present → None if any missing.
        4. Map claims to Identity:
           - id = payload[claim_mapping.id_claim]  # required
           - type = payload.get(claim_mapping.type_claim, "user")
           - roles = payload.get(claim_mapping.roles_claim, [])
           - attrs = {k: payload[k] for k in attrs_claims if k in payload}
        5. Return Identity(id=id, type=type, roles=roles, attrs=attrs).
        """

    def security_schemes(self) -> dict:
        return {"bearerAuth": {"type": "http", "scheme": "bearer", "bearerFormat": "JWT"}}
```

**Key resolution (CLI support):**
```python
# Priority: key_file > secret arg > JWT_SECRET env var
key = key_file_content or secret_arg or os.environ.get("JWT_SECRET")
```

**Algorithms**: Default `["HS256"]`. Supports `RS256`, `ES256` for asymmetric keys.

---

### 4. `AuthMiddleware` — `auth/middleware.py`

ASGI middleware that validates credentials and bridges `Identity` to downstream handlers.

```python
auth_identity_var: ContextVar[object | None] = ContextVar("auth_identity", default=None)

class AuthMiddleware:
    def __init__(
        self,
        app: ASGIApp,
        authenticator: Authenticator,
        *,
        exempt_paths: set[str] | None = None,      # exact path match, no auth check
        exempt_prefixes: set[str] | None = None,    # prefix match, no auth check
        require_auth: bool = True,
    ) -> None:
        self._app = app
        self._authenticator = authenticator
        self._exempt_paths = exempt_paths or {"/.well-known/agent.json", "/health", "/metrics"}
        self._exempt_prefixes = exempt_prefixes or set()
        self._require_auth = require_auth

    async def __call__(self, scope, receive, send) -> None:
        """ASGI middleware entry.

        1. Skip non-HTTP scopes (lifespan, websocket).
        2. Extract path; check if exempt_paths or exempt_prefixes match → skip auth.
        3. Extract headers dict (lowercase keys) from scope["headers"].
        4. identity = authenticator.authenticate(headers).
        5. If identity is None and require_auth:
           → send 401 with body {"error": "Authentication required"}
             and header WWW-Authenticate: Bearer.
        6. Set auth_identity_var.set(identity) (token for ContextVar).
        7. Await downstream app.
        8. Reset ContextVar in finally block.
        """
```

**Default exempt paths**: `{"/.well-known/agent.json", "/health", "/metrics"}`.

**Explorer exemption**: When `explorer=True`, add `explorer_prefix` to `exempt_prefixes`.

**Identity access in handlers:**
```python
from apcore_a2a.auth.middleware import auth_identity_var

# Inside any async route handler:
identity = auth_identity_var.get()  # Returns Identity or None
```

## Security Scheme Declaration in Agent Card

When auth is configured, `AgentCardBuilder` adds:
```json
{
  "securitySchemes": {
    "bearerAuth": {
      "type": "http",
      "scheme": "bearer",
      "bearerFormat": "JWT"
    }
  },
  "security": [{"bearerAuth": []}]
}
```

## File Structure

```
src/apcore_a2a/auth/
    __init__.py      # exports: Authenticator, JWTAuthenticator, ClaimMapping, AuthMiddleware,
                     #          auth_identity_var
    protocol.py      # Authenticator Protocol
    jwt.py           # JWTAuthenticator, ClaimMapping
    middleware.py    # AuthMiddleware, auth_identity_var
```

## Key Invariants

- `authenticate()` NEVER raises — returns None for any auth failure
- `AuthMiddleware` resets ContextVar in `finally` block (no identity leak between requests)
- `/.well-known/agent.json` is always exempt from auth (public discovery)
- `require_auth=False` (permissive mode): unauthenticated requests proceed with `identity=None`
- Identity flows through ContextVar, not function parameters (avoids invasive API changes)

## Test Module

`tests/auth/test_jwt_authenticator.py`
`tests/auth/test_auth_middleware.py`
