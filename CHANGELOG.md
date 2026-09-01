# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.6.0] - 2026-09-01

Minor release across all three SDKs, resolving `aiperceivable/apcore-a2a` issues #2,
#3, #4 and #5 and `apcore-a2a-rust` #2. One principle runs through all five: **apcore
already draws these distinctions, and a transport binding's job is to convey them,
not to flatten them.** Three governance refusals apcore names separately were being
collapsed onto two codes that mean something else; four behavioral annotations apcore
models were reaching no A2A caller at all; and the ACL's verdict on who may see what
was honoured in one binding, ignored in two.

The runtime floor moves to **apcore 0.28.0 / apcore-toolkit 0.10.2**, and the same
principle covers what apcore 0.28.0 changed underneath: it split one ACL verdict into
two independent results, and this release reads them apart rather than folding them
back together.

**Breaking** for callers that matched `-32001` or the literal `"Task not found"` to
detect an authorization failure, and for Python/TypeScript deployments that relied on
the public Agent Card as a full catalogue.

### Changed

- **A governance refusal is reported as itself** (srs FR-ERR-003, FR-ERR-009,
  FR-ERR-010, FR-ERR-012). `ACL_DENIED` moves from `-32001 "Task not found"` to
  `-32040 "Access denied"`; `APPROVAL_DENIED` and `APPROVAL_TIMEOUT` leave the
  `-32603` catch-all for `-32041 "Approval denied"` and `-32042 "Approval timed
  out"`. All three now reach `TASK_STATE_REJECTED` instead of `TASK_STATE_FAILED`,
  which matters most on `message/send`, where the response is a JSON-RPC `result`
  and the error code never reaches the caller at all.

  The old mapping told an agent a *different* failure had happened, one whose correct
  response was the opposite of the real one: `"Task not found"` sends a caller back to
  re-fetch or re-send the one thing that was fine, and `"Internal server error"` is
  the canonical *retryable* failure — for a call a human had explicitly refused. A2A
  §13.2 requires a server to return an authorization error (MUST) and to say what was
  refused (SHOULD); its MUST NOT forbids revealing *the existence of a resource*, not
  the *class* of failure, so a fixed `"Access denied"` naming no caller, target or rule
  satisfies all three. `-32040`–`-32042` sit above A2A 1.0's reserved `-32001`–`-32009`
  in the JSON-RPC implementation-defined range, which is the "JSON-RPC custom error"
  §13.2 names for this binding.

  `-32001` now means only "unknown task id, or a task owned by another principal" —
  still deliberately indistinguishable from each other, no longer conflated with
  authorization. `APPROVAL_PENDING` is untouched: it remains a resumable
  `TASK_STATE_INPUT_REQUIRED` carrying its message verbatim, which is how a caller
  learns the `approval_id` it resumes with.

- **The public Agent Card shows what an anonymous caller could actually invoke**
  (srs FR-AGC-003). Every registered skill, minus those the ACL denies to the
  anonymous principal, minus those annotated `requires_approval`. The three bindings
  previously gave three different answers: Rust filtered per caller, Python and
  TypeScript filtered nothing — publishing every restricted skill's id, name,
  description and full input schema to any anonymous caller, since `/.well-known/`
  is auth-exempt by design. None matched the static split the spec described.

  The filter resolves one identity, so it runs once at build time. Per-caller
  filtering moves to the extended card, where the caller is authenticated — which is
  what makes it affordable: on the auth-exempt public route it let any anonymous
  client drive one governance audit write per skill per request.

- **The extended Agent Card carries what the authenticated caller may invoke**
  (srs FR-AGC-004), including `requires_approval` skills. Python previously returned
  `CopyFrom(base_card)`, a verbatim copy; Rust and TypeScript advertised
  `capabilities.extendedAgentCard` and never routed the endpoint at all.

- **`capabilities.extendedAgentCard` is no longer derived from `auth is not None`**
  (srs FR-AGC-002, FR-AGC-006). A binding advertises a capability only when it serves
  it. A2A §3.2.x entitles a client to read the flag and call the method.

- **Card visibility reads apcore's two governance axes apart** (srs FR-AGC-003 "The
  two axes", criterion 11; FR-AGC-004 criteria 2 and 10). apcore 0.28.0
  (`PROTOCOL_SPEC` §6.1.6) gave an ACL rule an `approval: required` field orthogonal
  to `effect`, so one check now resolves two independent results — may this caller
  reach this target, and must this call be put to a human — and made the legacy
  boolean `ACL.check` **fail closed** on the second. All three bindings filtered
  their cards on that boolean. Left alone, a skill the ACL *allows* the caller but
  gates behind a human would have silently vanished from the **extended** card too:
  a refusal the ACL never issued, and the caller left unable to learn that a
  capability it holds exists at all.

  Every card filter now reads the structured accessor (`check_access` /
  `checkAccess`) and filters on `access` alone. `approval_required` decides only
  *which surface*: it joins the module's `requires_approval` annotation as the second
  source the public card subtracts, composed by union exactly as apcore §6.9 composes
  them. Reading the annotation alone would have left on the public card a skill an
  anonymous caller cannot in fact just call — since 0.28.0 the annotation describes
  the *module*, not the call (apcore#110).

  The ACL is consulted with no arguments projection, because a card is discovery and
  there is no call site. An `arguments` condition (§6.1.7) is therefore unevaluable,
  which leaves an `allow` rule's requirement *pending* and composing (§6.1.1 rule 5)
  — so a skill gated only for some argument shapes reports a requirement here and
  stays off the public card. That is the honest discovery-time answer.

### Added

- **apcore's behavioral annotations reach the wire** (srs FR-SKL-004). `readonly`,
  `destructive`, `idempotent` and `requires_approval` are emitted as namespaced
  entries in the standard `tags` field — `apcore:readonly`, `apcore:destructive`,
  `apcore:idempotent`, `apcore:requires-approval` — in that fixed order, appended
  after the module's own tags and de-duplicated against them. Only `true` flags are
  emitted, matching how the apcore MCP binding maps the same annotations onto
  optional `readOnlyHint` / `destructiveHint` / `idempotentHint`.

  A2A 1.0 `AgentSkill` is `{id, name, description, tags, examples, inputModes,
  outputModes, securityRequirements}` — no `extensions`, no `metadata`, and in the
  Python binding the type is generated from the A2A protobuf schema, so a vendor
  member cannot be added at all. `tags` is the only carrier that exists; the
  `apcore:` prefix keeps it out of the module's own flat tag namespace.

  This is also what makes retry semantics usable. `retryable` is a property of the
  *error*; whether a retry is safe is a property of the *operation*. A timeout is
  retryable for a read and dangerous for a non-idempotent mutation. apcore resolves
  that internally with exactly this two-factor rule; before this change an A2A caller
  had the error half and not the operation half.

- **`disclose_refusal_reason` configuration flag, default off** (srs FR-ERR-011).
  When enabled, forwards apcore's own sanitized message for the three governance
  codes instead of the fixed per-class string. The *class* of refusal is always
  conveyed; the *detail* is a deployment choice. A server whose callers are its own
  agents wants the reason — which is what the apcore MCP binding reports today, so an
  operator comparing the two transports otherwise sees it on one and not the other.

- **`FR-AGC-006`**: a binding SHALL advertise no capability it does not serve, with a
  conformance fixture asserting each advertised capability has a served counterpart.

- **apcore's `system.*` management namespace never reaches the public Agent Card**
  (srs FR-AGC-003 criteria 12 and 13, FR-AGC-004 criterion 11; `apcore-a2a#5`).
  Skills whose module id is in apcore's reserved namespace (`PROTOCOL_SPEC` §6.7)
  are removed from the public card **unconditionally** — independent of ACL state,
  of the `requires_approval` annotation, and of how `sys_modules` is configured.
  They stay on the extended card, filtered per identity like any other skill.

  Every other subtraction the public card makes is governance-shaped, and with no
  ACL configured they all collapse: the ACL predicates are empty, and the
  annotation covers only the three `system.control.*` write modules — leaving the
  six read modules, which enumerate the deployment's module inventory
  (`system.manifest.*`), its health and its usage, published by id, name,
  description and full input schema to any anonymous caller on the auth-exempt
  `/.well-known/` route. `ACL.discover()` yields nothing for a missing root by
  design, so "no ACL at all" is the default rather than an edge case, and the rule
  that has to hold there cannot be shaped like a governance verdict. Rust made
  this explicit — `let Some(acl) = executor.acl() else { return card.clone() }` —
  and Python and TypeScript reached the same place by filtering nothing.

  Keying on the namespace conveys apcore's own boundary rather than inventing one:
  apcore identifies the surface by that prefix itself, in
  `Executor.governance_state()`, and its registries reject a user module that
  tries to claim the namespace (`InvalidModuleId: Module ID contains reserved
  word: 'system'`), so the prefix cannot collide.

  Refusing to build such a card was the alternative and is deliberately **not**
  taken: it would break a running deployment on upgrade, and a binding cannot know
  the operator's intent. FR-AGC-007 carries the signal instead.

- **`FR-AGC-007`**: at server construction the binding reads apcore's
  `Executor.governance_state()` and warns when `unprotected_control_surface` is
  true. It never refuses to start and never alters a card.

  Withholding `system.*` from the public card removes the surface from *discovery*,
  not from *dispatch*. `system.control.*` declares `requires_approval`, but
  apcore's approval gate warns once and continues when no `ApprovalHandler` is
  configured, failing closed only under `ExecutionPolicy(strict=True)` — so the
  write modules stay callable, and the card rule must not be mistaken for a fix to
  that. `governance_state()` (apcore 0.28.0, `PROTOCOL_SPEC` §6.6.5) is the right
  question to ask: it answers "is a gate *engaging*", not "is an ACL *attached*",
  and the ACL and approval gates are pipeline *steps* that the `internal`,
  `testing` and `minimal` strategies remove — so an executor can hold an ACL that
  no step ever consults. apcore made it a pure read and named "a serve-time adapter
  may warn or refuse" as the two reactions; this is the warn.

### Runtime floor

The floor moves to **apcore 0.28.0 / apcore-toolkit 0.10.2**. apcore-toolkit 0.10.2
is a floor bump only — no API change. apcore 0.28.0 required one code change in this
project (the card-visibility axes, above) and one source fix in the Rust test suite
(`apcore::acl::ACLRule` gained a required `approval` field, which a struct literal
cannot skip). Three further apcore changes alter what an A2A caller observes without
any code change here, and are called out so an operator upgrading is not surprised:

- **Python only — a module declaring `input_schema` / `output_schema` as a raw dict
  is now actually validated.** `_DictSchemaAdapter.model_validate` was a
  pass-through, so every constraint such a schema declared (`required`, `enum`,
  `type`, …) was inert. Calls that previously reached the handler with junk now fail
  as `SCHEMA_VALIDATION_ERROR`, which this binding already maps to `-32602`. Modules
  using Pydantic models are unaffected.
- **An ACL `deny` rule whose condition cannot be evaluated now denies** instead of
  going inert. That changes both what the Agent Card advertises and what a call
  returns, in the safe direction.
- **An ACL rule carrying an `effect` outside `allow` / `deny` now fails at
  construction** rather than being read as `deny`. A deployment whose ACL file
  carried `effect: "Allow"` will fail to start rather than silently deny.

### Fixed

- **`sys_modules` / `sysModules` registered nothing, in all three bindings**
  (`apcore-a2a#5`). apcore reads `sys_modules.enabled` — a **top-level** config
  section — while Python and TypeScript built the registration `Config` as
  `{apcore: {sys_modules: {enabled: true}}}`, so the lookup missed,
  `register_sys_modules` returned at its first line, and Python logged
  `"Registered apcore system modules"` anyway. Rust never even tried: it passed
  `Config::default()` and discarded the `Result` with `let _ =`. The flag was a
  silent no-op in every deployment since it was introduced.

  Fixed together with the namespace rule above, deliberately in that order:
  repairing the config path on its own is precisely what would have opened the
  hole that rule closes. Rust now inspects the `Result` and reports a failure;
  Python and TypeScript carry through operator settings found under either
  spelling, top-level winning; all three log the module ids they actually
  registered rather than asserting success.

- **Rust: `async_serve` no longer derives its bind address from `config.url`**
  (`apcore-a2a-rust#2`). `APCoreA2AConfig` gains `host` and `port`; `url` is now used
  only for the Agent Card, defaulting to `http://{host}:{port}` exactly as the Python
  and TypeScript bindings have always done. The old code string-split `url` on `://`
  and fell back to `0.0.0.0:8000` when the split failed, so a scheme-less loopback
  value like `127.0.0.1:18999` silently published every skill on every interface, on
  a port the operator never chose — with nothing logged. A path, a trailing slash or a
  missing port failed to bind with an address-parse error naming neither the URL nor
  the cause. Rust also gains the unauthenticated-public-bind warning Python and
  TypeScript already emit.

### Conformance

- `fixtures/error_mapping.json`: `acl_denied_sanitized` re-pinned to `-32040`;
  `approval_denied_sanitized`, `approval_timeout_sanitized`,
  `approval_pending_is_not_a_refusal` and `task_not_found_keeps_minus_32001` added.
  The last two exist so the remap cannot silently take `APPROVAL_PENDING` or `-32001`
  with it.
- `fixtures/agent_card.json`: `annotation_tags_namespaced`,
  `annotation_tags_absent_when_false`, `public_card_hides_requires_approval`,
  `public_card_hides_acl_denied_to_anonymous`,
  `extended_card_is_not_a_copy_of_the_public_card`,
  `public_card_hides_acl_approval_gated` and `extended_card_keeps_acl_approval_gated`
  added — the card-visibility coverage whose absence let the three bindings diverge
  unnoticed, and the pair that pins both directions of the authorization/approval
  split. The last two carry `card_variant`, so builder-level runners skip them and
  each SDK's card-visibility test asserts them.
- `fixtures/agent_card.json`: `public_card_excludes_system_namespace_with_no_acl`
  and `extended_card_keeps_system_namespace` added — the "system modules enabled,
  no ACL" case `apcore-a2a#5` identified as the gap, asserted identically across
  the three bindings.

## [0.5.0] - 2026-08-17

Minor release across all three SDKs. Task-addressed methods are now scoped to the
authenticated principal, and failed tasks no longer collapse every error to a fixed
string — from `aiperceivable/apexe` issues #33, #34 and #35. The runtime floor moves
to apcore 0.27.

**Rust carries a breaking change**: task ownership moved inside the `TaskStore`, so
it survives a restart. Python and TypeScript needed no code change for that — both
re-export their upstream SDK's `TaskStore`, which already took a `ServerCallContext`
and already bucketed by owner. Suites: Rust 161, Python 350, TypeScript 328.

### Changed

- **Task-addressed methods are scoped to the authenticated principal** in all three
  SDKs — `tasks/get`, `tasks/cancel`, `tasks/list` and the three
  `tasks/pushNotificationConfig/*`. Previously `tasks/list` returned every caller's
  tasks including their output, and any task could be read, cancelled or
  re-pointed at a webhook by whoever knew its id; only the unguessability of a
  UUIDv4 stood in the way. Cross-principal access is masked as `-32001` so ids
  cannot be probed, consistent with how ACL denials are already reported
  (srs FR-ERR-003). Callers with no identity share one owner bucket, matching
  upstream's `UnauthenticatedUser`.

- **Failed tasks classify their errors instead of collapsing them.** Caller-fixable
  failures — schema validation, invalid input, unknown module — now reach the caller
  with their code, message and `ai_guidance`, which is what lets an agent on the
  other end correct itself. Internal and unrecognized errors keep the fixed
  `"Internal server error"` string and ACL denials stay masked, per the existing
  `error_mapping.json` contract, which is unchanged.

- **`features/storage.md`: owner scoping is now part of the `TaskStore` contract.**
  Every method must be scoped by the call context; `get` and `delete` on another
  owner's task must behave exactly as they do for a task that does not exist — never
  a distinguishable error, or task ids become probeable — and `list` must return only
  the caller's tasks. Bucketing on `(owner, task_id)` satisfies all of it at once.
  No language can enforce this: upstream `a2a-python` states it as a SHOULD, and a
  store that ignores its context disables isolation entirely. Recorded alongside is
  the split of push-notification configs into their own pluggable store, so a
  persistent deployment restores tasks and their webhook targets together.

  Rust is the only SDK that defines these traits itself (there is no Rust a2a-sdk),
  so it is where the contract is implemented; its `TaskStore` and `PushConfigStore`
  signatures in this doc were updated to match.

- **Runtime floor raised to apcore 0.27** in all three SDKs. Note for future
  upgrades: 0.27's own breaking-change list is incomplete — `Registry::describe`
  changed both its signature *and* its return value (a module's one-line description
  became a whole Markdown document), which would have published the full document
  into every `AgentSkill.description` on the Agent Card.

- **`tasks/list` renamed to `ListTasks` throughout (srs, prd, tech-design,
  test-plan, features).** The spec had required a method name that exists in no
  A2A version: 1.0 calls task listing `ListTasks`, and 0.3 had no listing method
  at all. Both SDK-backed implementations therefore could not serve it, while
  the Rust server implemented the invented name and nothing else could reach it.
  `srs.md` now also records the `A2A-Version: 1.0` header requirement, and why
  the remaining method names stay in their 0.3 spellings.

  FR-TSK-006's pagination fields were wrong in the same way: it specified
  `cursor` / `limit` / `nextCursor`, while `ListTasksRequest` declares
  `pageToken` / `pageSize` and answers with `nextPageToken` / `totalSize`. The
  `pageSize <= 200` clamp it required is recorded as not implemented — no
  upstream SDK enforces a ceiling, and the Rust server ignores list parameters
  altogether (no `contextId` filter, no pagination).

### Fixed

- **`features/storage.md` corrected against the implementations.** Four statements
  did not match any SDK: the Rust `TaskStore` signature predated the owner-carrying
  call context; the "actual implementation" snippet re-exported from a `self::memory`
  module that does not exist (the types are defined in `storage/mod.rs`); the a2a-sdk
  interface was described as `save`/`get`/`delete` only, while 1.0 also has `list`
  and passes a `ServerCallContext` to each; and `isinstance(store, TaskStore)` was
  presented as a cross-language requirement when it is Python-only.

- **Rust API surface brought back in sync** across `features/public-api.md`,
  `features/server-core.md` and `spec/tech-design.md`: `A2AServerFactory::create`
  takes a `CreateOptions` struct (was ten positional arguments); the crate-root
  re-exports now list the `storage` types and `CreateOptions`; and `VERSION` is
  derived from `CARGO_PKG_VERSION` rather than the hand-written `"0.4.0"` literal
  the docs still showed — that literal had already drifted twice, which is why the
  constant became derived in the first place.

- **`features/push-notifications.md` no longer describes one implementation for
  three languages.** Python and TypeScript delegate config storage, delivery and
  retry to their upstream SDK and cannot inject a store; Rust defines its own
  pluggable `PushConfigStore` and delivers itself. The note previously described
  only the Python path.

- **`spec/tech-design.md` §13.2 (Package Manifest) rebuilt from the shipped
  manifests.** All three samples had drifted far past their version headings: the
  Python one still read `version = "0.1.0"` with `apcore>=0.22.0` and
  `a2a-sdk>=0.3.0`, and omitted `apcore-toolkit` entirely; the TypeScript one listed
  `@a2a-js/sdk: ^0.3.0` against a shipped floor of `1.0.1`, plus `express ^4.19`
  against `^5.1.0`, and omitted `apcore-js`; the Rust one listed `apcore = "0.22"`
  and eight of its twenty-three dependencies. All 53 dependency lines across the
  three samples are now verified equal to the manifests, and the section states that
  the repo manifests are authoritative. Added the reasoning behind two pins that
  look arbitrary otherwise: why `@a2a-js/sdk` is floored at `>=1.0.1` rather than
  `^1.0.0`, and why `apcore` is the one dependency carrying an upper bound.

- **The SSE wire format is documented as the one all three SDKs actually emit.**
  `features/streaming.md`, `spec/srs.md` and both example blocks in
  `spec/tech-design.md` showed a bare `data: {"statusUpdate":…}` payload preceded
  by a monotonically increasing SSE `id:` line. No SDK produces either. Every
  frame is a complete JSON-RPC response whose `result` holds the event, with `id`
  echoing the `message/stream` request, and no `id:` line at all — upstream
  `a2a-python`'s SSE generator sets only `data` (plus `event: error` on the error
  frame), and the Rust `sse_event` never calls `.id()`. A client written against
  the old text could not have parsed a real stream.

- **The `oneof` wrapper key for a standalone message is `message`, not `msg`.**
  Corrected in `features/streaming.md` and `spec/srs.md` (prose and the
  wrapper-key table). Verified against the upstream proto descriptor:
  `StreamResponse` maps `task -> task`, `message -> message`,
  `status_update -> statusUpdate`, `artifact_update -> artifactUpdate`.

- **`spec/tech-design.md` §13.3 (Docker) — two samples that could not have worked.**
  The TypeScript Dockerfile ran `npm ci` against a `package-lock.json`; the repo
  uses pnpm and ships only `pnpm-lock.yaml`. The Python one ran `pip install .`
  before `COPY src/ src/`, which hatchling cannot satisfy — its wheel target is
  `packages = ["src/apcore_a2a"]`.

## [0.4.2] - 2026-06-25

Patch release. Bumps the required apcore runtime floor to 0.25.0 and apcore-toolkit to 0.9.1 across all three SDKs. No code, API, or wire-protocol changes — the A2A 1.0 contract is unchanged. All suites pass unmodified: Python 332, TypeScript 306, Rust 106.

### Changed

- **Runtime floor bumped** — `apcore >= 0.25.0` (from `>=0.24.0`) and `apcore-toolkit >= 0.9.1` (from `>=0.8.1`) in all SDKs (Rust: `apcore = "0.25"`). The adapter surface is unaffected by the 0.24 → 0.25 delta.

  apcore 0.25.0 and apcore-toolkit 0.9.0–0.9.1 changes reviewed for adapter impact — none required a change:
  - **Config-driven ACL discovery (0.25.0, apcore #74)** — `ACL.discover(config)` is auto-wired during `APCore` construction, but is skipped when the caller supplies its own `Executor` (as all three SDK adapters do), so an explicitly configured ACL is never clobbered.
  - **Registry module-id constants promoted to the public surface (0.25.0, apcore #30)** — export-surface-only addition; no behavior change.
  - **apcore-toolkit OpenAPI parser hardening (0.9.0–0.9.1)** — robustness fixes (integer status-code keys, explicit-`null` fields) with no public API change; the adapters use only `deep_resolve_refs` / `deepResolveRefs`, which is unaffected.


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
