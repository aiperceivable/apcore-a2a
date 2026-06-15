---
description: "CLI module spec: the apcore-a2a command-line entry point with a serve subcommand that discovers modules from an extensions dir and launches an A2A server without writing code."
---

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

## File: `__main__.py` (Python) / `cli.ts` (TypeScript) / `cli.rs` (Rust)

The Python CLI exposes a `serve` subcommand with a rich flag set; the TypeScript
CLI mirrors it (`npx apcore-a2a serve ...`); the Rust CLI is a thin clap `Cli`
struct with no `serve` subcommand and only a minimal set of flags (auth,
explorer, metrics, and similar options are configured via code/env instead).

=== "Python"

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
            "--host", default="127.0.0.1",
            help="Bind host (default: 127.0.0.1)",
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

=== "TypeScript"

    The TypeScript CLI (`src/cli.ts`, bin `apcore-a2a`) registers a `serve`
    subcommand whose flags mirror the Python set: `--extensions-dir` (required),
    `--host` (127.0.0.1), `--port` (8000), `--name`, `--description`,
    `--version-str`, `--url`, `--auth-type bearer`, `--auth-key`,
    `--auth-issuer`, `--auth-audience`, `--push-notifications`, `--explorer`,
    `--cors-origins`, `--execution-timeout` (300), `--log-level` (info), and
    `--metrics`. It exports `main()` and `resolveAuthKey()`.

    ```typescript
    // src/cli.ts — entry point, invoked as `npx apcore-a2a serve ...`
    export async function main(): Promise<void> {
      // parse flags (commander), build the registry from --extensions-dir,
      // resolve auth via resolveAuthKey(), then call serve(registry, opts).
    }
    ```

=== "Rust"

    The Rust CLI is a clap `Cli` struct with no `serve` subcommand. Only
    `-e/--extensions-dir`, `-n/--name`, `--url`, and `-p/--port` are exposed.
    Auth, explorer, metrics, CORS, push notifications, execution timeout, and
    log level are not CLI flags — configure them via code/env instead.

    ```rust
    #[derive(Parser)]
    #[command(name = "apcore-a2a", version, about = "A2A protocol adapter for apcore")]
    pub struct Cli {
        #[arg(short, long, default_value = "./extensions")] pub extensions_dir: String,
        #[arg(short, long, default_value = "apcore-a2a")] pub name: String,
        #[arg(long, default_value = "http://localhost:8000")] pub url: String,
        #[arg(short, long, default_value_t = 8000)] pub port: u16,
    }

    pub fn run() -> Result<(), Box<dyn std::error::Error>> {
        // parse Cli, build the backend from extensions_dir, then call serve(..).
    }
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

**Step 4b — Security warning:**
```python
if args.host == "0.0.0.0" and auth is None:
    logger.warning(
        "--host 0.0.0.0 binds to all network interfaces without authentication; "
        "consider using --host 127.0.0.1 or enabling --auth-type bearer"
    )
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

=== "Python"

    ```python
    def _resolve_auth_key(auth_key: str | None) -> str | None:
        """Resolve auth key: file path > direct value > APCORE_JWT_SECRET env var.

        Priority order:
        1. If auth_key is a path to an existing file → read file contents.
        2. If auth_key is provided and not a file path → use as literal key.
        3. If auth_key is None → check os.environ.get("APCORE_JWT_SECRET").
        4. Return None if all sources empty.
        """
        if auth_key:
            p = Path(auth_key)
            if p.exists():
                return p.read_text().strip()
            return auth_key
        return os.environ.get("APCORE_JWT_SECRET")
    ```

=== "TypeScript"

    The TypeScript CLI exports `resolveAuthKey(authKey?)` with the same priority
    order: an existing file path is read; otherwise the value is used literally;
    otherwise it falls back to the `APCORE_JWT_SECRET` env var.

    ```typescript
    // src/cli.ts
    export function resolveAuthKey(authKey?: string): string | undefined {
      // file path → read; else literal; else process.env.APCORE_JWT_SECRET
    }
    ```

=== "Rust"

    > Not applicable — the Rust CLI exposes no auth flags, so there is no
    > `resolve_auth_key` helper. Configure JWT auth in code via
    > `JWTAuthenticator::new(std::env::var("APCORE_JWT_SECRET")?)` and the
    > `*_with_auth` entry points instead.

---

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Clean shutdown (SIGINT/SIGTERM/KeyboardInterrupt) |
| `1` | Configuration error (invalid args, missing dir, zero modules, missing auth key) |
| `2` | Runtime error (unrecoverable server crash) |

---

## CLI Invocation Examples

=== "Python"

    ```bash
    # Minimal
    apcore-a2a serve --extensions-dir ./extensions

    # With auth
    apcore-a2a serve --extensions-dir ./ext --auth-type bearer --auth-key $APCORE_JWT_SECRET

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
      --auth-key $APCORE_JWT_SECRET \
      --auth-issuer https://auth.example.com \
      --push-notifications \
      --explorer \
      --cors-origins "https://app.example.com" \
      --log-level info

    # Show version
    apcore-a2a --version
    ```

=== "TypeScript"

    ```bash
    # Minimal
    npx apcore-a2a serve --extensions-dir ./extensions

    # With auth
    npx apcore-a2a serve --extensions-dir ./ext --auth-type bearer --auth-key $APCORE_JWT_SECRET

    # With auth key file
    npx apcore-a2a serve --extensions-dir ./ext --auth-type bearer --auth-key /run/secrets/jwt.key

    # Full configuration
    npx apcore-a2a serve \
      --extensions-dir ./extensions \
      --host 0.0.0.0 \
      --port 8080 \
      --name "My Agent" \
      --description "Agent for image processing" \
      --auth-type bearer \
      --auth-key $APCORE_JWT_SECRET \
      --auth-issuer https://auth.example.com \
      --push-notifications \
      --explorer \
      --cors-origins "https://app.example.com" \
      --metrics \
      --log-level info
    ```

=== "Rust"

    The Rust binary has no `serve` subcommand and exposes only
    `-e/--extensions-dir`, `-n/--name`, `--url`, and `-p/--port`. Auth, explorer,
    push notifications, CORS, and metrics are not CLI flags (configure them via
    code/env instead).

    ```bash
    # Minimal
    apcore-a2a --extensions-dir ./extensions

    # Name + port (port is folded into the bind address; --url sets the card URL)
    apcore-a2a \
      --extensions-dir ./extensions \
      --name "My Agent" \
      --url http://localhost:8080 \
      --port 8080

    # Show version (clap-generated)
    apcore-a2a --version
    ```

---

## Entry Point Registration

Each SDK registers the `apcore-a2a` binary through its own packaging manifest.

=== "Python"

    ```toml
    # pyproject.toml
    [project.scripts]
    apcore-a2a = "apcore_a2a.__main__:main"
    ```

    Also invocable as a module:

    ```bash
    python -m apcore_a2a serve --extensions-dir ./ext
    ```

=== "TypeScript"

    ```json
    // package.json
    {
      "bin": { "apcore-a2a": "dist/cli.js" }
    }
    ```

    Invocable via the package runner:

    ```bash
    npx apcore-a2a serve --extensions-dir ./ext
    ```

=== "Rust"

    ```toml
    # Cargo.toml
    [[bin]]
    name = "apcore-a2a"
    path = "src/bin/apcore-a2a.rs"
    ```

    Build/run the binary with Cargo:

    ```bash
    cargo run --bin apcore-a2a -- --extensions-dir ./ext
    ```

---

## File Structure

=== "Python"

    ```
    src/apcore_a2a/
        __main__.py    # main(), _run_serve(), _resolve_auth_key()
    ```

=== "TypeScript"

    ```
    src/
        cli.ts         # main(), resolveAuthKey()
    ```

=== "Rust"

    ```
    src/
        cli.rs              # Cli (clap Parser), run()
        bin/apcore-a2a.rs   # binary entry point
    ```

## Key Invariants

- All validation errors print to `stderr` and exit with code 1 (not 2)
- Runtime errors exit with code 2
- Clean shutdown exits with code 0
- `--auth-key` accepts file path or literal string or falls back to `APCORE_JWT_SECRET` env var
- No modules discovered → exit 1 immediately (don't start server with empty card)
- Extensions dir must exist AND be a directory (two separate checks, distinct messages)

## Test Module

`tests/test_cli.py`
