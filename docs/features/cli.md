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
        # Not `required=True` since 0.7.0: `--from-openapi` is the alternative
        # backend source. One of the two is required, and supplying neither is a
        # usage error (exit 2), which is what argparse produced when this flag
        # carried `required=True`.
        serve_parser.add_argument(
            "--extensions-dir", default=None,
            help="Path to directory containing apcore module extensions",
        )
        # --- OpenAPI backend (F-12), 0.7.0 ---
        serve_parser.add_argument(
            "--from-openapi", default=None, dest="from_openapi",
            help="OpenAPI 3.0/3.1 spec URL or path; every operation becomes a Skill",
        )
        serve_parser.add_argument(
            "--openapi-base-url", default=None, dest="openapi_base_url",
            help="Base URL for proxied requests (default: the document's servers[0].url)",
        )
        serve_parser.add_argument(
            "--openapi-prefix", default=None, dest="openapi_prefix",
            help="Prepended to every derived module ID; required alongside --extensions-dir",
        )
        serve_parser.add_argument(
            "--openapi-include", default=None, dest="openapi_include",
        )
        serve_parser.add_argument(
            "--openapi-exclude", default=None, dest="openapi_exclude",
        )
        serve_parser.add_argument(
            "--openapi-header", action="append", default=None,
            dest="openapi_headers", metavar="KEY:VALUE",
            help="Header for the spec fetch only; repeatable. Never sent on proxied calls",
        )
        serve_parser.add_argument(
            "--openapi-no-deprecated", action="store_true",
            dest="openapi_no_deprecated",
            help="Skip operations marked deprecated: true",
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
    subcommand whose flags mirror the Python set: `--extensions-dir`,
    `--host` (127.0.0.1), `--port` (8000), `--name`, `--description`,
    `--version-str`, `--url`, `--auth-type bearer`, `--auth-key`,
    `--auth-issuer`, `--auth-audience`, `--push-notifications`, `--explorer`,
    `--cors-origins`, `--execution-timeout` (300), `--log-level` (info),
    `--metrics`, and the seven F-12 flags (`--from-openapi`,
    `--openapi-base-url`, `--openapi-prefix`, `--openapi-include`,
    `--openapi-exclude`, `--openapi-header`, `--openapi-no-deprecated`). It
    exports `main()`, `resolveAuthKey()` and `parseOpenapiHeaders()`.

    ```typescript
    // src/cli.ts — entry point, invoked as `npx apcore-a2a serve ...`
    export async function main(): Promise<void> {
      // parse flags (commander), build the registry from --extensions-dir,
      // resolve auth via resolveAuthKey(), then call serve(registry, opts).
    }
    ```

=== "Rust"

    The Rust CLI is a clap `Cli` struct with no `serve` subcommand. It exposes
    `-e/--extensions-dir`, `-n/--name`, `--url`, `-p/--port` and the seven F-12
    `--openapi-*` flags. Auth, explorer, metrics, CORS, push notifications,
    execution timeout, and log level are not CLI flags — configure them via
    code/env instead.

    Two 0.7.0 changes: `--extensions-dir` lost its `./extensions` default and is
    now `Option`, since `--from-openapi` is an alternative source and a bare
    invocation should not silently serve a directory the operator never named;
    and **`--port` was inert before 0.7.0** — `run()` built its config with
    `..Default::default()` and never copied the parsed value, so the server bound
    `0.0.0.0:8000` whatever was typed. `--url` now follows `--port` as well
    (defaulting to `http://localhost:<port>`), because fixing the bind alone
    would have left the Agent Card advertising a socket the server does not bind.

    ```rust
    #[derive(Parser)]
    #[command(name = "apcore-a2a", version, about = "A2A protocol adapter for apcore")]
    pub struct Cli {
        #[arg(short, long)] pub extensions_dir: Option<String>,
        #[arg(long)] pub from_openapi: Option<String>,
        #[arg(long)] pub openapi_prefix: Option<String>,
        // ... --openapi-base-url / -include / -exclude / -header / -no-deprecated
        #[arg(short, long, default_value = "apcore-a2a")] pub name: String,
        #[arg(long)] pub url: Option<String>,
        #[arg(short, long, default_value_t = 8000)] pub port: u16,
    }

    pub async fn run() -> Result<(), Box<dyn std::error::Error>> {
        // parse Cli, build the backend from extensions_dir and/or from_openapi
        // into ONE registry (so the collision preflight sees both), then serve.
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

| Code | Meaning | Retryable? |
|------|---------|---|
| `0` | Clean shutdown (SIGINT/SIGTERM/KeyboardInterrupt) | — |
| `1` | **Configuration or runtime error.** The command line was well-formed; the environment it named was not — a missing extensions directory, zero modules discovered, an unresolvable OpenAPI spec, a missing auth key, an unreachable spec host | **Yes.** The world can become ready |
| `2` | **Usage error.** The command line itself is wrong — no backend source named, an unknown flag, a flag missing its value, a malformed `--openapi-header` | **No.** Retrying it unchanged will always fail |

All three bindings implement both tiers, and `2` is the code `argparse`, `clap` and GNU
`getopt` all use for a usage fault, so the bindings agree with each other and with the tools
around them.

!!! note "This was a genuine defect until 0.7.0, not a language difference"
    Measured before the fix: Python exited `2` for both usage faults (argparse) and `1` for a
    configuration fault — correct. TypeScript collapsed everything onto `1`, losing the
    distinction entirely. **Rust disagreed with itself**: `apcore-a2a --bogus` exited `2`
    because clap caught it, while `apcore-a2a` with no backend source exited `1` because the
    hand-written check did — the same class of mistake, two codes, decided by which layer
    noticed first.

    The tiers are worth keeping apart because a supervisor or CI job can act on them: retry on
    `1`, never on `2`. Collapsing them onto one code throws that away, and aligning *downward*
    to `1` would have meant overriding `ArgumentParser.error()` so that Python's unknown-flag
    handling diverged from every other Python CLI.

### Unrecognized arguments

All three bindings refuse an argument the flag table does not define, and exit `2`.

!!! danger "TypeScript silently ignored them until 0.7.0, and the failure widened exposure"
    Measured, with a one-letter typo:

    ```
    apcore-a2a serve --extensions-dir ./ext --openapi-exclde 'secret.*'
      values:      { "openapi-exclde": true, … }   ← the flag became a boolean
      positionals: [ "serve", "secret.*" ]         ← its value became a positional
    ```

    The exclusion did not apply, and every operation it was meant to hold back was scanned,
    registered, and published to the **public Agent Card** — a route served without
    authentication and built to be crawled. The server started and reported success.

    Python (`unrecognized arguments`) and Rust (`unexpected argument … found`) both refused the
    same argv and exited `2`, so the tolerant behaviour was never portable: a deployment could
    only have depended on it in TypeScript, and the command it depended on was already failing
    in the other two bindings.

    Fixed by an explicit check, **not** by `strict: true`. `strict: false` is kept deliberately
    — it is what lets the CLI tolerate `--metrics=x` for a boolean and a repeat of a
    non-`multiple` option, and `strict: true` would turn both into throws. The check is scoped
    to the case that is actually dangerous: a spelling no binding recognises.

!!! note "argparse additionally accepts unambiguous abbreviations"
    `--openapi-exclud` (one letter short) is a legal spelling in Python, because argparse
    resolves unambiguous prefixes. That is an argparse feature, not a parity requirement —
    clap does not do it, and TypeScript does not either. The shared contract is only that a
    spelling *no* binding recognises is refused rather than ignored.

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

    # Serve an OpenAPI document instead of an extensions directory (F-12)
    apcore-a2a serve \
      --from-openapi https://petstore3.swagger.io/api/v3/openapi.json \
      --openapi-prefix petstore

    # Both sources at once — --openapi-prefix is REQUIRED here, or a derived
    # module ID can collide with a project module ID
    apcore-a2a serve \
      --extensions-dir ./extensions \
      --from-openapi ./openapi.json \
      --openapi-prefix petstore \
      --openapi-header "X-Api-Key: $PETSTORE_SPEC_KEY" \
      --openapi-no-deprecated

    # Show version
    apcore-a2a --version
    ```

=== "TypeScript"

    ```bash
    # Minimal
    npx apcore-a2a serve --extensions-dir ./extensions

    # OpenAPI document as the backend source (F-12)
    npx apcore-a2a serve \
      --from-openapi https://petstore3.swagger.io/api/v3/openapi.json \
      --openapi-prefix petstore

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
