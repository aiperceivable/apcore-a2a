# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.4.1] - 2026-06-15

Patch release. Bumps the required apcore runtime floor to 0.24.0 and apcore-toolkit to 0.8.1 across all three SDKs. No code, API, or wire-protocol changes — the A2A 1.0 contract is unchanged. All suites pass unmodified: Python 332, TypeScript 306, Rust 106.

### Changed

- **Runtime floor bumped** — `apcore >= 0.24.0` (from `>=0.22.0`) and `apcore-toolkit >= 0.8.1` (from `>=0.8.0`) in all SDKs (Rust: `apcore = "0.24"`, whose previous `"0.22"` caret hard-capped below 0.23 and required the bump). The adapter surface is unaffected by the 0.22 → 0.24 delta.

  apcore 0.23.0–0.24.0 changes reviewed for adapter impact — none required a change:
  - **Per-instance `ToggleState` (0.24.0, apcore #71)** — `Executor` / `register_sys_modules` gained an optional per-instance toggle state; all SDK call sites use the back-compat form and fall back to the process-global toggle state, behaviorally identical for a single-registry server.
  - **`CircuitBreakerMiddleware` constructor rewrite (0.23.0, breaking)** — not used by the adapter.
  - **AI error-recovery metadata auto-populated on `ModuleError` (0.23.0)** — `user_fixable` / `ai_guidance` now surface through serialized errors automatically; the adapter never backfilled them.
  - **`A2ASubscriber` 4xx no-retry (0.23.0)** — applies to apcore's own event-system subscriber, not this adapter.


## [0.4.0] - 2026-06-01

### Changed

- **A2A protocol upgraded 0.3 → 1.0 (BREAKING).** All language SDKs now target the protobuf-derived A2A 1.0 wire format:
  - **Events** are a `oneof` discriminated by wrapper key (`task` / `statusUpdate` / `artifactUpdate` / `msg`) — the 0.3 `type`/`kind` discriminator and the `final` flag are removed. The stream now ends on a terminal `TaskState` (`TASK_STATE_COMPLETED` / `TASK_STATE_FAILED` / `TASK_STATE_CANCELED` / `TASK_STATE_REJECTED`).
  - **`TaskState`** serializes as its full enum name (`TASK_STATE_SUBMITTED`, …); `Role` is an enum (`ROLE_USER` / `ROLE_AGENT`).
  - **`Part`** is a flattened `oneof`: text → `{"text": …}`, data → `{"data": …}`, file → `{"raw": …}` / `{"url": …}` (no `type` discriminator).
  - **`AgentCard`**: 0.3 top-level `url` → `supportedInterfaces` (`[{url, protocolBinding:"JSONRPC", protocolVersion:"1.0", tenant:""}]`); `supportsAuthenticatedExtendedCard` → `capabilities.extendedAgentCard`; `capabilities` gains `extensions`, drops `stateTransitionHistory`; `AgentSkill` gains `securityRequirements`; new required `provider`, `securityRequirements`, `signatures`.
  - **Agent Card endpoint** is `/.well-known/agent-card.json` (1.0), with `/.well-known/agent.json` served as a 0.3 compatibility alias.
- **apcore runtime bumped to 0.22** across all SDKs; `apcore-toolkit >= 0.8.0` added as a dependency (schema `$ref` resolution now delegates to the shared `deep_resolve_refs`).
- **New apcore 0.22 capabilities wired:** real streaming via `Executor.stream()` (`StreamingModule`), cooperative cancellation via `CancelToken`, `global_deadline` (mapped from `execution_timeout`, now bounding the streaming path), `ObsLoggingMiddleware`, and `register_sys_modules` (new `sys_modules` option on `serve()` / `async_serve()`).
- **Env prefix simplified** — `APCORE__A2A` (double underscore) → `APCORE_A2A` (single underscore).

### Added

- **Rust implementation** (`apcore-a2a-rust`) — a working A2A 1.0 server on axum 0.8 over apcore 0.22, at full feature parity with the Python and TypeScript SDKs (hand-rolled A2A 1.0 wire types; there is no Rust A2A SDK).
- **Error Formatter Registry** (§8.8) — all SDKs register their `ErrorMapper` with `ErrorFormatterRegistry.register("a2a", ...)` so the ecosystem has a discoverable A2A error formatter.
- **Config Bus namespace** (§9.13) — all SDKs register the `apcore-a2a` namespace with env prefix `APCORE_A2A` and defaults for `execution_timeout`, `cors_origins`, `explorer`, `metrics`, `push_notifications`.
- **New error codes** — `MODULE_DISABLED`, `CONFIG_NAMESPACE_DUPLICATE`, `CONFIG_MOUNT_ERROR`, `CONFIG_BIND_ERROR` handled in ErrorMapper across all languages.
- **Cross-language conformance fixtures** (`conformance/`) — shared golden cases for agent card shape, error mapping, JWT claim coercion, part conversion, skill resolution, and streaming events, consumed by the Python/Rust/TypeScript conformance suites to lock A2A 1.0 behavior across languages.
- **`LICENSE`** (Apache-2.0) and a repo `.gitignore`.

### Fixed (Cross-Language Sync)

- **TypeScript top-level exports** — `index.ts` now exports all 16 symbols specified in F-08 (was 6). Added `VERSION`, `createAuthMiddleware`, `authIdentityStore`, `getAuthIdentity`, `AgentCardBuilder`, `SkillMapper`, `SchemaConverter`, `ErrorMapper`, `PartConverter`, `A2AServerFactory`, `ApCoreAgentExecutor`.
- **TypeScript `pushNotifications` parameter** — added to `A2AServerCreateOptions` and wired into capabilities.
- **TypeScript `A2AClient.agentCard`** getter — equivalent to Python's `agent_card` async property.
- **TypeScript `ErrorMapper.sanitizeMessage`** — changed from public to private, matching Python and spec.

### Notes

- All three implementations are upgraded to this spec and passing: Python (`a2a-sdk >= 1.0.0`, 282 tests), TypeScript (`@a2a-js/sdk >= 1.0.0-alpha.0`, 257 tests), and Rust (`apcore-a2a-rust`, 37 tests). The Rust server reaches full feature parity: JSON-RPC (message/send, message/stream SSE, tasks/get|cancel|list, pushNotificationConfig/set|get|delete), CancelToken, global_deadline, JWT auth, ObsLogging, sys_modules, CORS, Explorer UI, and webhook push delivery.

---

## [0.3.0] - 2026-03-27

### Added

- **Display overlay in `SkillMapper`** (§5.13) — both Python and TypeScript SDKs now read `metadata["display"]["a2a"]` for skill name, description, tags, and guidance.
  - Skill name: `a2a.alias` → `display.alias` → humanized `module_id`.
  - Description: `a2a.description` → `display.description` → `module.description`. Guidance appended if present.
  - Tags: `display.tags` → `module.tags`.

### Changed

- **Cross-language sync** — aligned Python and TypeScript SDKs on endpoint, env vars, CLI interface, and documentation.
- **Well-known endpoint** unified to `/.well-known/agent.json` across both SDKs (TypeScript was using `agent-card.json`).
- **Environment variables** renamed with `APCORE_` prefix: `JWT_SECRET` → `APCORE_JWT_SECRET`, `A2A_EXECUTION_TIMEOUT` → `APCORE_A2A_EXECUTION_TIMEOUT`.
- **CLI `--execution-timeout`** now accepts seconds in both SDKs (TypeScript was using milliseconds).
- **`apcore` dependency** bumped from `0.9.0+` to `0.14.0+` in both SDKs.
- Updated feature spec `docs/features/adapters.md` — SkillMapper field mapping table corrected (`id`→`module_id`, `name`→alias chain).

### Removed

- **`_build_extensions()` dead code** (Python) — `AgentSkill` has no `extensions` field in the A2A SDK; deleted along with 3 tests.

---

## [0.2.1] - 2026-03-22

### Changed
- Rebrand: aipartnerup → aiperceivable

## [0.2.0] - 2026-03-10

### Changed

- Updated Python requirement from 3.10+ to 3.11+ to match pyproject.toml.
- Updated apcore dependency requirement from 0.6.0+ to 0.9.0+.
- Delegated TaskStore and InMemoryTaskStore to a2a-sdk (replaces custom protocol).
- Replaced ExecutionRouter, TaskManager, and TransportManager with a2a-sdk components (ApCoreAgentExecutor, DefaultRequestHandler, A2AStarletteApplication).
- CLI `--host` default changed from `0.0.0.0` to `127.0.0.1` for safer defaults.
- Default agent name fallback changed from `"apcore-agent"` to `"Apcore Agent"`.
- Auth 401 response body now returns `{"error": "Unauthorized", "detail": "Missing or invalid Bearer token"}`.
- AgentCard `defaultOutputModes` now includes both `"text/plain"` and `"application/json"`.

### Added

- Expanded public API exports: auth classes, adapter classes, and server factory now re-exported from top-level `__init__.py`.
- Documented that SkillMapper `_build_extensions()` cannot wire annotations into AgentSkill (a2a-sdk lacks `extensions` field); annotations available via Explorer UI instead.
- ErrorMapper `_sanitize_message()` now strips traceback lines in addition to file paths.
- Explorer `create_explorer_mount()` accepts optional `registry` parameter to enrich agent card with input schemas.
- Path-based registry resolution: `serve()` and `async_serve()` accept `str`/`Path` for auto-discovery.

### Fixed

- Feature specs updated to match actual implementation (F-01 through F-11).
- Documentation version references corrected (Python 3.11+, apcore 0.9.0+).

## [0.1.0] - 2026-03-07

### Added

- Product Requirements Document (PRD), Software Requirements Specification (SRS), Technical Design Document, and Test Plan.
- 11 feature specifications: adapters, storage, server-core, streaming, push-notifications, auth, client, public-api, cli, explorer, ops.
- Getting Started guide with Python and TypeScript examples.
- README with project overview, quick start, and architecture summary.
- MkDocs Material documentation site with GitHub Pages deployment.
- Project logo (`apcore-a2a-logo.svg`).
