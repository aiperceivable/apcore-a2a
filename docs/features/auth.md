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

=== "Python"

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

=== "TypeScript"

    ```typescript
    // src/auth/types.ts
    export interface Authenticator {
      // Validate credentials from lowercase-keyed HTTP headers.
      // Returns an Identity on success, null on invalid/missing credentials.
      authenticate(headers: Record<string, string>): Identity | null;
      // A2A-compatible security scheme dict for Agent Card declaration, e.g.
      // { bearerAuth: { type: "http", scheme: "bearer", bearerFormat: "JWT" } }
      securitySchemes(): Record<string, unknown>;
    }
    ```

=== "Rust"

    ```rust
    // src/auth/protocol.rs
    use async_trait::async_trait;
    use serde_json::Value;
    use std::collections::HashMap;

    #[async_trait]
    pub trait Authenticator: Send + Sync {
        // Validate credentials from lowercase-keyed HTTP headers.
        // Returns Some(Identity) on success, None on invalid/missing credentials.
        async fn authenticate(&self, headers: &HashMap<String, String>) -> Option<Identity>;
        // A2A-compatible security scheme JSON for Agent Card declaration, e.g.
        // { "bearerAuth": { "type": "http", "scheme": "bearer", "bearerFormat": "JWT" } }
        fn security_schemes(&self) -> Option<Value>;
    }
    ```

**Runtime checkable**: `isinstance(auth, Authenticator)` returns True for any duck-typed implementation.

---

### 2. `ClaimMapping` — `auth/jwt.py`

=== "Python"

    ```python
    from dataclasses import dataclass, field

    @dataclass
    class ClaimMapping:
        id_claim: str = "sub"          # JWT claim → Identity.id
        type_claim: str = "type"       # JWT claim → Identity.type
        roles_claim: str = "roles"     # JWT claim → Identity.roles (list[str])
        attrs_claims: list[str] = field(default_factory=list)  # extra claims → Identity.attrs
    ```

=== "TypeScript"

    ```typescript
    // src/auth/types.ts — all fields optional (camelCase)
    export interface ClaimMapping {
      idClaim?: string;        // JWT claim → Identity.id     (default "sub")
      typeClaim?: string;      // JWT claim → Identity.type   (default "type")
      rolesClaim?: string;     // JWT claim → Identity.roles  (default "roles")
      attrsClaims?: string[];  // extra claims → Identity.attrs (default [])
    }
    ```

=== "Rust"

    ```rust
    // src/auth/jwt.rs — fields are snake_case; use ..Default::default() for omitted ones
    pub struct ClaimMapping {
        pub id_claim: String,        // JWT claim → Identity.id     (default "sub")
        pub type_claim: String,      // JWT claim → Identity.type   (default "type")
        pub roles_claim: String,     // JWT claim → Identity.roles  (default "roles")
        pub attrs_claims: Vec<String>, // extra claims → Identity.attrs (default [])
    }
    ```

---

### 3. `JWTAuthenticator` — `auth/jwt.py`

=== "Python"

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
            4. Map claims to Identity using the canonical scalar-coercion rule (below):
               - id   = coerce(payload[id_claim])      # required; None → reject token
               - type = coerce(payload[type_claim]) or "user"   # null/non-scalar → "user"
               - roles = [coerce(r) for r in payload[roles_claim] if scalar]  # non-scalar dropped
               - attrs = {k: payload[k] for k in attrs_claims if k in payload}
            5. Return Identity(id=id, type=type, roles=roles, attrs=attrs).
            """

        def security_schemes(self) -> dict:
            return {"bearerAuth": {"type": "http", "scheme": "bearer", "bearerFormat": "JWT"}}
    ```

=== "TypeScript"

    ```typescript
    // Constructor: key first, all options in a single object arg.
    new JWTAuthenticator(key: string, opts?: JWTAuthenticatorOptions)

    interface JWTAuthenticatorOptions {
      algorithms?: jwt.Algorithm[];   // Default: ["HS256"]
      audience?: string;
      issuer?: string;
      claimMapping?: ClaimMapping;
      requireClaims?: string[];       // Default: ["sub"]
    }

    // authenticate(headers): validates the Bearer JWT and applies the same
    //   canonical scalar-coercion rule (below). Returns an Identity or null;
    //   never throws.
    // securitySchemes(): returns
    //   { bearerAuth: { type: "http", scheme: "bearer", bearerFormat: "JWT" } }
    ```

=== "Rust"

    ```rust
    // Constructor takes only the secret; options are applied via chained builders.
    impl JWTAuthenticator {
        pub fn new(secret: impl Into<String>) -> Self;
        pub fn with_claim_mapping(mut self, mapping: ClaimMapping) -> Self;
        pub fn with_require_claims(mut self, claims: Vec<String>) -> Self;  // Default: ["sub"]
        pub fn with_algorithms(mut self, algorithms: Vec<Algorithm>) -> Self; // Default: [HS256]
        pub fn with_audience(mut self, audience: impl Into<String>) -> Self;
        pub fn with_issuer(mut self, issuer: impl Into<String>) -> Self;
    }

    // Authenticator impl:
    //   authenticate(headers): validates the Bearer JWT and applies the same
    //     canonical scalar-coercion rule (below). Returns Some(Identity) or None;
    //     never panics.
    //   security_schemes(): returns
    //     { "bearerAuth": { "type": "http", "scheme": "bearer", "bearerFormat": "JWT" } }
    ```

**Key resolution (CLI support):**

=== "Python"

    ```python
    # Priority: key_file > secret arg > APCORE_JWT_SECRET env var
    key = key_file_content or secret_arg or os.environ.get("APCORE_JWT_SECRET")
    ```

=== "TypeScript"

    ```typescript
    // resolveAuthKey(authKey?): file path → read file; else literal value;
    //   else APCORE_JWT_SECRET env var.
    const key = resolveAuthKey(authKey);
    ```

=== "Rust"

    > Not applicable — the Rust CLI exposes no auth flags, so there is no
    > key-resolution helper. Construct `JWTAuthenticator::new(...)` directly with a
    > secret read from your own source (e.g. `std::env::var("APCORE_JWT_SECRET")`).

**Algorithms**: Default `["HS256"]`. Supports `RS256`, `ES256` for asymmetric keys.

**Claim coercion (canonical cross-language rule).** A claim value is coerced to a
string identically across all SDKs:

| Claim value | Coerced to |
|---|---|
| string | the string itself |
| number | its string form (`12345` → `"12345"`) |
| boolean | `"true"` / `"false"` (lowercase) |
| `null`, array, object | **rejected** (no valid string) |

Applied per field:
- **`id`** (`sub`): a non-scalar value is **not** a valid identity — the token is
  rejected (`authenticate()` returns `None`). A scalar numeric/boolean `sub` is
  coerced. (PyJWT's RFC `sub`-must-be-string check is disabled so a scalar numeric
  `sub` reaches this coercion instead of being rejected at decode time.)
- **`type`**: an absent, `null`, or non-scalar value falls back to `"user"`; an
  explicit empty string is preserved.
- **`roles`**: only when the claim is an array; non-scalar elements are dropped,
  scalars are coerced.

This is enforced identically in Python, TypeScript, and Rust so the same token yields
the same `Identity` (or the same rejection) in every SDK.

---

### 4. `AuthMiddleware` — `auth/middleware.py`

ASGI middleware that validates credentials and bridges `Identity` to downstream handlers.

=== "Python"

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
            self._exempt_paths = exempt_paths or {"/.well-known/agent-card.json", "/.well-known/agent.json", "/health", "/metrics"}
            self._exempt_prefixes = exempt_prefixes or set()
            self._require_auth = require_auth

        async def __call__(self, scope, receive, send) -> None:
            """ASGI middleware entry.

            1. Skip non-HTTP scopes (lifespan, websocket).
            2. Extract path; check if exempt_paths or exempt_prefixes match → skip auth.
            3. Extract headers dict (lowercase keys) from scope["headers"].
            4. identity = authenticator.authenticate(headers).
            5. If identity is None and require_auth:
               → send 401 with body {"error": "Unauthorized", "detail": "Missing or invalid Bearer token"}
                 and header WWW-Authenticate: Bearer.
            6. Set auth_identity_var.set(identity) (token for ContextVar).
            7. Await downstream app.
            8. Reset ContextVar in finally block.
            """
    ```

=== "TypeScript"

    ```typescript
    // Express middleware factory. Identity is stashed in an AsyncLocalStorage
    // instead of a ContextVar.
    createAuthMiddleware(opts: {
      authenticator: Authenticator;
      exemptPaths?: Set<string>;      // exact path match, no auth check
      exemptPrefixes?: Set<string>;   // prefix match, no auth check
      requireAuth?: boolean;          // default true
    }): ExpressMiddleware

    // Default exempt paths:
    //   "/.well-known/agent-card.json", "/.well-known/agent.json", "/health", "/metrics".
    // On null identity + requireAuth → responds 401
    //   { error: "Unauthorized", detail: "Missing or invalid Bearer token" }
    //   with header WWW-Authenticate: Bearer.
    export const authIdentityStore: AsyncLocalStorage<Identity | null>;
    ```

=== "Rust"

    ```rust
    // Axum tower layer. Identity is exposed via the AUTH_IDENTITY request extension.
    AuthMiddlewareLayer::new(
        authenticator: Arc<dyn Authenticator>,
        exempt_paths: Vec<String>,
    ) -> Self  // strict (require_auth = true)

    AuthMiddlewareLayer::with_require_auth(
        authenticator: Arc<dyn Authenticator>,
        exempt_paths: Vec<String>,
        require_auth: bool,
    ) -> Self

    pub const AUTH_IDENTITY: &str = "auth_identity";
    // On None identity + require_auth → responds 401
    //   { "error": "Unauthorized", "detail": "Missing or invalid Bearer token" }
    //   with header WWW-Authenticate: Bearer.
    ```

**Default exempt paths**: `{"/.well-known/agent-card.json", "/.well-known/agent.json", "/health", "/metrics"}`.

**Explorer exemption**: When `explorer=True`, add `explorer_prefix` to `exempt_prefixes`.

**Identity access in handlers:**

=== "Python"

    ```python
    from apcore_a2a.auth.middleware import auth_identity_var

    # Inside any async route handler:
    identity = auth_identity_var.get()  # Returns Identity or None
    ```

=== "TypeScript"

    ```typescript
    import { getAuthIdentity } from "apcore-a2a";

    // Inside any request handler (within the AsyncLocalStorage scope):
    const identity = getAuthIdentity();  // Returns Identity or null
    ```

=== "Rust"

    ```rust
    use apcore_a2a::AUTH_IDENTITY;

    // Inside an axum handler, read the identity from request extensions:
    let identity = req.extensions().get::<Identity>();  // Option<&Identity>
    ```

## Security Scheme Declaration in Agent Card

`JWTAuthenticator.security_schemes()` returns the flat descriptor
`{"bearerAuth": {"type": "http", "scheme": "bearer", "bearerFormat": "JWT"}}`.
When building the card, `AgentCardBuilder` converts each descriptor into the A2A 1.0
protobuf-JSON **`oneof`** shape (the same shape the reference a2a-sdk serializes), so
the served card carries:
```json
{
  "securitySchemes": {
    "bearerAuth": {
      "httpAuthSecurityScheme": {
        "scheme": "bearer",
        "bearerFormat": "JWT"
      }
    }
  },
  "securityRequirements": []
}
```

> The scheme type is encoded by the `oneof` key (`httpAuthSecurityScheme`), not a
> `type` field — this is the canonical A2A 1.0 wire shape and is byte-identical
> across the Python/TypeScript/Rust SDKs. `securityRequirements` is currently always
> `[]` (empty) in all three SDKs.

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
- `/.well-known/agent-card.json` (and its `/.well-known/agent.json` alias) is always exempt from auth (public discovery)
- `require_auth=False` (permissive mode): unauthenticated requests proceed with `identity=None`
- Identity flows through ContextVar, not function parameters (avoids invasive API changes)

## Test Module

`tests/auth/test_jwt_authenticator.py`
`tests/auth/test_auth_middleware.py`
