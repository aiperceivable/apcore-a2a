# Feature: CLI Module

| Field | Value |
|-------|-------|
| Feature ID | F-09 |
| Name | cli |
| Priority | P1 |
| SRS Refs | FR-CMD-001, FR-CMD-002 |
| Tech Design | §4.9 CLI Module |
| Depends On | F-08 (public-api) |
| Blocks | None |

## Purpose

Command-line interface for launching an A2A agent server without writing Python code. Discovers modules from an extensions directory, configures auth/push/explorer from flags, then calls `serve()`. Entry point registered as `apcore-a2a` in `pyproject.toml`.

## File: `__main__.py`

```python
def main() -> None:
    parser = argparse.ArgumentParser(
        prog="apcore-a2a",
        description="Launch an A2A agent server from apcore modules",
    )
    parser.add_argument(
        "--version", action="version",
        version=f"%(prog)s {__version__}",
    )

    subparsers = parser.add_subparsers(dest="command")

    # --- serve subcommand ---
    serve_parser = subparsers.add_parser("serve", help="Start A2A server")
    serve_parser.add_argument(
        "--extensions-dir", required=True,
        help="Path to directory containing apcore module extensions",
    )
    serve_parser.add_argument(
        "--host", default="0.0.0.0",
        help="Bind host (default: 0.0.0.0)",
    )
    serve_parser.add_argument(
        "--port", type=int, default=8000,
        help="Bind port (default: 8000)",
    )
    serve_parser.add_argument(
        "--name", default=None,
        help="Agent display name (default: from registry config)",
    )
    serve_parser.add_argument(
        "--description", default=None,
        help="Agent description",
    )
    serve_parser.add_argument(
        "--version-str", default=None, dest="agent_version",
        help="Agent version string (default: from registry config)",
    )
    serve_parser.add_argument(
        "--url", default=None,
        help="Public base URL for Agent Card (default: http://{host}:{port})",
    )
    serve_parser.add_argument(
        "--auth-type", choices=["bearer"], default=None,
        help="Authentication type",
    )
    serve_parser.add_argument(
        "--auth-key", default=None,
        help="JWT verification key or path to key file",
    )
    serve_parser.add_argument(
        "--auth-issuer", default=None,
        help="Expected JWT issuer (iss claim)",
    )
    serve_parser.add_argument(
        "--auth-audience", default=None,
        help="Expected JWT audience (aud claim)",
    )
    serve_parser.add_argument(
        "--push-notifications", action="store_true",
        help="Enable push notification support",
    )
    serve_parser.add_argument(
        "--explorer", action="store_true",
        help="Enable Explorer UI",
    )
    serve_parser.add_argument(
        "--cors-origins", nargs="*", default=None,
        help="Allowed CORS origins (space-separated)",
    )
    serve_parser.add_argument(
        "--execution-timeout", type=int, default=300,
        help="Task execution timeout in seconds (default: 300)",
    )
    serve_parser.add_argument(
        "--log-level",
        choices=["debug", "info", "warning", "error"],
        default="info",
        help="Logging level (default: info)",
    )

    args = parser.parse_args()

    if args.command == "serve":
        _run_serve(args)
    else:
        parser.print_help()
        sys.exit(1)


def _run_serve(args: argparse.Namespace) -> None:
    """Validate args, build Registry, configure auth, call serve()."""
    ...


if __name__ == "__main__":
    main()
```

---

## `_run_serve()` Logic

**Step 1 — Validate extensions dir:**
```python
extensions_dir = Path(args.extensions_dir).resolve()
if not extensions_dir.exists():
    print(f"Error: Extensions directory not found: {extensions_dir}", file=sys.stderr)
    sys.exit(1)
if not extensions_dir.is_dir():
    print(f"Error: Not a directory: {extensions_dir}", file=sys.stderr)
    sys.exit(1)
```

**Step 2 — Load Registry:**
```python
from apcore import Registry
registry = Registry(extensions_dir=str(extensions_dir))
modules = registry.list()
if not modules:
    print(f"Error: No modules discovered in {extensions_dir}", file=sys.stderr)
    sys.exit(1)
print(f"Discovered {len(modules)} module(s): {', '.join(modules)}")
```

**Step 3 — Validate and build auth:**
```python
auth = None
if args.auth_type == "bearer":
    key = _resolve_auth_key(args.auth_key)
    if not key:
        print("Error: --auth-key is required when --auth-type is bearer", file=sys.stderr)
        sys.exit(1)
    from apcore_a2a.auth import JWTAuthenticator
    auth = JWTAuthenticator(
        key=key,
        issuer=args.auth_issuer,
        audience=args.auth_audience,
    )
```

**Step 4 — Resolve URL:**
```python
url = args.url or f"http://{args.host}:{args.port}"
```

**Step 5 — Call `serve()`:**
```python
from apcore_a2a import serve
try:
    serve(
        registry,
        host=args.host,
        port=args.port,
        name=args.name,
        description=args.description,
        version=args.agent_version,
        url=url,
        auth=auth,
        push_notifications=args.push_notifications,
        explorer=args.explorer,
        cors_origins=args.cors_origins,
        execution_timeout=args.execution_timeout,
        log_level=args.log_level,
    )
except KeyboardInterrupt:
    sys.exit(0)
except Exception as e:
    print(f"Error: {e}", file=sys.stderr)
    sys.exit(2)
```

---

## `_resolve_auth_key()` — Key Resolution

```python
def _resolve_auth_key(auth_key: str | None) -> str | None:
    """Resolve auth key: file path > direct value > JWT_SECRET env var.

    Priority order:
    1. If auth_key is a path to an existing file → read file contents.
    2. If auth_key is provided and not a file path → use as literal key.
    3. If auth_key is None → check os.environ.get("JWT_SECRET").
    4. Return None if all sources empty.
    """
    if auth_key:
        p = Path(auth_key)
        if p.exists():
            return p.read_text().strip()
        return auth_key
    return os.environ.get("JWT_SECRET")
```

---

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Clean shutdown (SIGINT/SIGTERM/KeyboardInterrupt) |
| `1` | Configuration error (invalid args, missing dir, zero modules, missing auth key) |
| `2` | Runtime error (unrecoverable server crash) |

---

## CLI Invocation Examples

```bash
# Minimal
apcore-a2a serve --extensions-dir ./extensions

# With auth
apcore-a2a serve --extensions-dir ./ext --auth-type bearer --auth-key $JWT_SECRET

# With auth key file
apcore-a2a serve --extensions-dir ./ext --auth-type bearer --auth-key /run/secrets/jwt.key

# Full configuration
apcore-a2a serve \
  --extensions-dir ./extensions \
  --host 0.0.0.0 \
  --port 8080 \
  --name "My Agent" \
  --description "Agent for image processing" \
  --auth-type bearer \
  --auth-key $JWT_SECRET \
  --auth-issuer https://auth.example.com \
  --push-notifications \
  --explorer \
  --cors-origins "https://app.example.com" \
  --log-level info

# Show version
apcore-a2a --version
```

---

## Entry Point Registration (`pyproject.toml`)

```toml
[project.scripts]
apcore-a2a = "apcore_a2a.__main__:main"
```

Also invocable as a module:
```bash
python -m apcore_a2a serve --extensions-dir ./ext
```

---

## File Structure

```
src/apcore_a2a/
    __main__.py    # main(), _run_serve(), _resolve_auth_key()
```

## Key Invariants

- All validation errors print to `stderr` and exit with code 1 (not 2)
- Runtime errors exit with code 2
- Clean shutdown exits with code 0
- `--auth-key` accepts file path or literal string or falls back to `JWT_SECRET` env var
- No modules discovered → exit 1 immediately (don't start server with empty card)
- Extensions dir must exist AND be a directory (two separate checks, distinct messages)

## Test Module

`tests/test_cli.py`
