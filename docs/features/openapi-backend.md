---
description: "OpenAPI Backend feature spec: a fourth apcore-a2a backend source that turns an OpenAPI 3.0/3.1 document into A2A Skills via apcore-toolkit's OpenAPIScanner and HTTPProxyRegistryWriter, including skill mapping, the empty-description fallback, the public Agent Card exposure of unapproved write operations, ACL target stability, the path-typed spec key, and SSRF boundaries."
---

# Feature: OpenAPI Backend

| Field | Value |
|-------|-------|
| Feature ID | F-12 |
| Name | openapi-backend |
| Priority | P1 |
| SRS Refs | FR-OAS-001 … FR-OAS-006 |
| Tech Design | §4.10 OpenAPI Backend |
| Depends On | F-08 (public-api — the backend-source entry points), F-01 (adapters — AgentCardBuilder/SkillMapper) |
| Blocks | None |

> Source: apcore-toolkit 0.11.0 OpenAPI Scanner
> ([`openapi-scanner.md`](https://github.com/aiperceivable/apcore-toolkit/blob/main/docs/features/openapi-scanner.md)
> § Phase 3 — Consumers), extended to apcore-a2a. apcore-cli shipped the same scanner as
> **FE-15a OpenAPI Import** in its 0.12.0, and apcore-mcp as the **OpenAPI Backend** in its 0.20.0.
> This is the third and last consumer in the ecosystem.
> Created: 2026-09-07

## Purpose

apcore-a2a turns an apcore project into an A2A agent. The OpenAPI Backend widens the input side of
that sentence: point the adapter at an **OpenAPI 3.0/3.1 document** and every operation in it becomes
an A2A Skill on the Agent Card, proxied over HTTP to the API that published the document — with no
apcore project on the other end at all.

The whole path already exists in shipped code. apcore-toolkit 0.11.0 added `OpenAPIScanner`, which
turns a document into `ScannedModule`s with a byte-identical `module_id` derivation across all three
SDKs; `HTTPProxyRegistryWriter` has shipped since 0.7 and registers those modules as executable HTTP
proxies. What is missing is the last link: a backend source that composes the two into a `Registry`
and hands it to the machinery apcore-a2a already has. This feature is that link and nothing more —
it introduces no scanning logic, no schema conversion, and no new execution path.

```
load_spec(url|path) → OpenAPIScanner.scan() → HTTPProxyRegistryWriter.write() → Registry
                                                                                  │
                                            existing adapter, unchanged ──────────┘
                                            (SkillMapper → SchemaConverter → AgentCardBuilder →
                                             card visibility → ACL → Approval → Executor)
```

## Scope

**Included:**

- A backend-source entry point per SDK that takes a spec location and returns a populated `Registry`,
  plus the wiring that lets `serve()` / `async_serve()` / `build_app()` accept it.
- The `apcore-a2a.openapi` Config Bus section and the corresponding CLI flags.
- The **empty-description fallback** (FR-OAS-003) — an A2A-specific requirement with no counterpart
  in the other two consumers.
- The **unapproved-write warning** (FR-OAS-005) and its interaction with the public Agent Card.
- Startup reporting of scan warnings, description fallbacks, and write failures.

**Excluded:**

- The scanner and the writer themselves. Both live in apcore-toolkit; their behaviour, the
  `module_id` derivation algorithm, and the `metadata` execution contract (`http_method` /
  `url_path`) are specified in apcore-toolkit's `openapi-scanner.md` and pinned by that
  repository's 24-case conformance corpus. This spec does not restate them.
- Swagger 2.0. `OpenAPIScanner.scan` refuses anything that is not `3.0.x` / `3.1.x`.
- Obtaining credentials for the upstream API. The writer's `auth_header_factory` hook is where a
  token arrives; producing one is the deployment's problem.
- Generating source code. Static mode persists a reviewable `.binding.yaml`, not stubs.

## Core Responsibilities

1. **Compose, do not reimplement.** The entry point calls `load_spec`, `OpenAPIScanner.scan` and
   `HTTPProxyRegistryWriter.write` in that order and returns the resulting `Registry`. Every
   downstream stage — skill mapping, schema conversion, card construction, ACL filtering, approval,
   execution — is the code that already serves an extensions directory, unmodified.
2. **Make the silences audible.** An OpenAPI document describes an API's *shape*. It says almost
   nothing about the *consequences* of calling an operation, and nothing at all about who may. The
   backend must not let that silence read as safety — see [Governance](#governance).
3. **Lose no operation without saying so.** Two filters downstream of the scanner can drop an
   operation from the Agent Card: a missing description (§ [Skill mapping](#skill-mapping)) and the
   card-visibility rules (§ [Governance](#governance)). Both must report what they dropped.

---

## Backend sources

apcore-a2a has three backend sources today. This adds a fourth:

| Source | Produces | Since |
|---|---|---|
| A `Registry` supplied by the caller | itself | 0.1 |
| An `Executor` supplied by the caller | itself (no `Registry`, so `sys_modules` is unavailable) | 0.1 |
| An extensions directory | `Registry` via apcore discovery | 0.1 |
| **An OpenAPI document** | **`Registry` via scan + write** | **0.7.0** |

The new source produces a `Registry`, not an `Executor`, which is deliberate: `register_sys_modules`
requires a `Registry`, so returning one keeps `sys_modules` available to an OpenAPI-backed server
exactly as it is for an extensions directory.

**Mixing sources.** An OpenAPI source and an extensions directory may be combined, but only when
`prefix` is set — otherwise a derived `module_id` can collide with a project module ID and the
collision preflight (below) refuses to start. Supplying both a `Registry`/`Executor` and an OpenAPI
source is refused: the caller already holds the object the backend would build.

## Skill mapping

A derived `module_id` reaches the Agent Card through the existing `SkillMapper` with no special
casing — an OpenAPI-derived module is a module. `metadata["display"]["a2a"]` overrides still apply
(§5.13), which is the supported route for renaming a skill whose upstream `operationId` is
unreadable.

Two things happen between the scanner and the card, and **both are the backend's job**: a derived ID
must be projected into apcore's registry alphabet, and an empty description must be repaired.
Without the first the registry is empty; without the second the card is silently short.

### Module ID projection (FR-OAS-002)

!!! danger "Without this step the canonical Swagger Petstore yields an empty Agent Card"
    The two alphabets do not agree, and the gap is not an edge case:

    - apcore-toolkit's `derive_module_id` sanitizes an operation into `[A-Za-z0-9_.-]` — mixed case
      and hyphens survive.
    - apcore's `Registry` accepts only `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`, enforced at
      `register` and again at `Executor.call`.

    Measured against apcore 0.30.0 / apcore-toolkit 0.11.1: **only two of nine realistic operation
    shapes register unrepaired, and the canonical Swagger Petstore is entirely in the rejected set.**
    It scans cleanly, produces a full `ScannedModule` list, registers nothing, and the server starts
    and serves an Agent Card with zero skills. Nothing in the path raises.

**Requirement.** Before registration, every derived `module_id` MUST be projected:

1. lowercase;
2. `-` → `_`;
3. if every dot-separated segment then matches `^[a-z][a-z0-9_]*$`, use the projected ID; otherwise
   the module is **dropped** — the ID cannot be repaired without inventing one.

A dropped module MUST be reported at WARNING naming both the derived ID and the offending segment
(e.g. `2fa`, which cannot be repaired because an apcore segment may not begin with a digit). This
report is the backend's responsibility and cannot be delegated: the projection runs inside the
scanner's `transform_module` hook, and a hook returning `None` drops the module **silently**.

**Two orderings are normative**, because implementing this in three SDKs produces two answers to
each:

- **A caller's own `transform_module` hook runs first, the projection last.** That way "every
  registered module ID is apcore-legal" holds unconditionally, whatever the caller's hook returns.
- **The projection runs before the scanner's `deduplicate_ids`,** because lowercasing can *create* a
  collision the document did not have: `listPets` and `listpets` are two distinct operations to
  OpenAPI and one module ID to apcore. Projecting afterwards would register one and lose the other
  with no warning; projecting first lets the scanner's own deduplication rename the second to
  `listpets_2` and attach a warning.

This is an apcore-registry constraint, not an A2A one — apcore-mcp and apcore-cli carry the
identical projection, and the behaviour is pinned by the shared fixture below so the three
consumers cannot drift.

### The empty-description fallback

**This is the one place the A2A binding must add behaviour the other two consumers did not need.**

`OpenAPIScanner` derives a module's description as:

```
description = summary or first_line(description) or ""
```

An operation carrying neither `summary` nor `description` therefore yields the empty string. And
`AgentCardBuilder` **skips any module whose description is empty or whitespace-only** — a rule that
predates this feature, is asserted by tests in all three SDKs, and exists for a good reason: a skill
with no description is not discoverable in any useful sense, and A2A clients rank skills by it.

Composed naively, those two correct behaviours produce a silent failure: a document with 80
operations, 30 of them undocumented, yields an Agent Card with 50 skills and **no indication that 30
were dropped**. The operator sees a working server advertising a subset of their API, and nothing
says which subset or why.

**Requirement (FR-OAS-003).** A scanned module whose description is empty or whitespace-only MUST
receive a synthesized description before the module reaches the registry, of the form:

```
{METHOD} {url_path}
```

taken from the `http_method` / `url_path` metadata keys the writer already requires — e.g.
`GET /pets/{petId}`. The synthesized value is a factual statement of what the operation is, carries
no invented semantics, and is stable across scans of the same document.

The backend MUST report the count and the affected module IDs at startup, at INFO:

```
apcore-a2a: 30 of 80 scanned operations had no summary or description;
            a "{METHOD} {path}" description was synthesized so they appear on the Agent Card.
            Affected: petstore.pets.pet_id.delete, petstore.store.inventory.get, ...
```

**Why synthesize rather than drop.** Dropping is the status quo and it is the failure this
requirement exists to close. Refusing to start is disproportionate — an undocumented operation is
extremely common in real documents and is not a safety problem. Synthesizing puts the operation on
the card where the operator can see it, and the INFO line tells them exactly which parts of their
document need `summary` fields. The synthesized form is deliberately ugly so that it reads as a
placeholder rather than as documentation.

**Interaction with `include`/`exclude`.** The fallback runs *after* the scanner's own filters, so an
operation excluded by configuration is never synthesized for and never reported.

### ID collisions are a startup failure

Two operations deriving the same `module_id`, or a derived ID colliding with a project module from a
combined extensions directory, MUST fail at startup naming both sources. It MUST NOT be resolved by
last-write-wins, which would silently shadow one operation with another, nor by suffixing, which
would produce an ID no ACL rule was written against.

The preflight runs before any module is registered, so a collision never leaves a half-populated
registry behind.

---

## Governance

This is the part of the feature that is not mechanical, and the A2A binding has one exposure the
other two consumers do not.

### Every scanned module arrives with `requires_approval = false`

The toolkit infers annotations from the HTTP method alone (RFC 9110 §9.2 safe-method semantics):

| Method | Inferred annotations | apcore-a2a skill tags |
|---|---|---|
| `GET` | `readonly`, `cacheable` | `apcore:readonly` |
| `HEAD`, `OPTIONS` | `readonly` | `apcore:readonly` |
| `PUT` | `idempotent` | `apcore:idempotent` |
| `DELETE` | `destructive` | `apcore:destructive` |
| **`POST`, `PATCH`, `TRACE`** | **none** | none |

`open_world` defaults to `true`, which is correct — these modules do call an external system.

`requires_approval` is **never** inferred, for any method. A `POST /charges` that moves money is
annotated exactly like a `POST /echo`, and the approval gate does not fire for either. The scanner
cannot know which operations are consequential, and guessing from the method would be wrong in both
directions — most `POST`s are harmless, some `GET`s are not — so it declines to guess.

### The public Agent Card advertises them anyway

!!! danger "An unapproved write operation reaches the public Agent Card by default"
    This is the A2A-specific consequence, and it is strictly worse than the equivalent in the other
    two consumers.

    Since 0.6.0 the **public** Agent Card is the anonymous-filtered card: it subtracts skills an
    `@external` caller may not invoke, and skills carrying an approval requirement (FR-AGC-003).
    A scanned module satisfies neither subtraction — the ACL is absent or permissive by default, and
    `requires_approval` is `false` by construction. So `POST /charges` appears on
    `/.well-known/agent-card.json`, a route that is **auth-exempt by design** and that A2A clients
    are built to crawl.

    apcore-mcp's equivalent exposure is a tool list behind a session; apcore-cli's is a local
    command surface. A2A's is a public discovery document. The same missing annotation therefore
    costs more here, which is why this section states a requirement the other two specs do not.

**The skills are not withheld as a class.** This is a deliberate departure from the `system.*` rule
(FR-AGC-003 criterion 12), and the distinction is worth stating because the two look similar:

- `system.*` is **apcore's own management surface**, identified by a prefix apcore reserves and
  `Registry::register_module` refuses to let a user module claim. Withholding it takes nothing from
  the operator, because it was never theirs to publish.
- An OpenAPI-derived module is **the operator's own API**. Withholding it as a class would be the
  binding overriding the intent of someone who pointed the adapter at a document precisely so it
  would be served. A binding that silently serves half of what it was given is the failure mode
  FR-OAS-003 above exists to close; re-introducing it here for a different reason would be
  incoherent.

**So the answer is a warning, not a filter** — the same shape as FR-AGC-007's unprotected-control-surface
guard, and on the same predicate family.

### FR-OAS-005: the unapproved-write warning

The backend MUST log a WARNING at startup when it registers any module whose HTTP method is in
`{POST, PUT, PATCH, DELETE}` and no approval path is demonstrable. The message MUST name the public
Agent Card, because that is the exposure that distinguishes this binding:

```
apcore-a2a: 12 scanned write operations (POST/PUT/PATCH/DELETE) have no approval requirement
            and will be advertised on the PUBLIC Agent Card at /.well-known/agent-card.json,
            which is served without authentication. Configure an ACL rule carrying
            `approval: required`, set requires_approval via a transform_module hook, or set
            apcore-a2a.openapi.acknowledge_unapproved_writes: true to record this as intended.
```

!!! danger "An attached ACL is not proof that anything will ask for approval"
    apcore's own `GovernanceState.unprotected_control_surface` states the rule this warning follows:
    it reports the **absence of a gate, never the presence of protection** — a wired ACL that
    permits every call still yields `False`.

    **The warning is suppressed only by:**

    - no module with a method in `{POST, PUT, PATCH, DELETE}` in the scanned set; or
    - every such module declaring `requires_approval` itself — reachable only through a
      `transform_module` hook, since the scanner never infers it; or
    - an explicit operator acknowledgement, `apcore-a2a.openapi.acknowledge_unapproved_writes: true`,
      which is a recorded decision rather than an inference.

    **It is never suppressed by `governance_state().acl_configured`.** `default_effect: allow`, an
    ACL whose `targets` never match these modules, or an ACL that allows without carrying
    `approval: required` anywhere each leaves every write operation ungated while satisfying "an ACL
    is attached". The shape of a rule is decidable; what it matches is not, and a predicate that
    cannot be closed must not be used to silence a safety warning.

    **It is escalated** when `governance_state().builtin_approval_gate_wired` is `false` — under the
    `internal`, `testing` and `minimal` strategies the approval gate is not in the pipeline at all,
    so neither a module-level `requires_approval` nor an ACL rule's `approval: required` would fire.

!!! note "Why there is no ACL-aware middle tier"
    An earlier draft of this section had a third tier that softened the wording when an ACL carried
    at least one `approval: required` rule. It was removed during implementation for two reasons,
    both worth recording so it is not reintroduced.

    First, **the field it named does not exist.** Measured against apcore 0.30.0, `GovernanceState`
    carries `control_modules_registered`, `read_modules_registered`, `acl_configured`,
    `builtin_acl_gate_wired`, `approval_handler_configured`, `builtin_approval_gate_wired`,
    `policy_strict`, `all_control_modules_require_approval` and `unprotected_control_surface` — and
    nothing reporting whether any rule carries an approval requirement.

    Second, and decisively, **the backend could not answer the question even if the field existed.**
    It builds the `Registry` that the `Executor` is later constructed *from*, so at the moment this
    warning fires there is no `Executor` and no ACL to inspect. The predicate is unknowable at this
    point, not merely unimplemented.

    That question belongs to **FR-AGC-007**'s serve-time warning, which runs where the `Executor`
    does exist and already reads `governance_state()`. The two warnings are complementary: this one
    reports what the *document* declares, FR-AGC-007 reports what the *deployment* enforces. The
    base wording says "no **module-level** approval requirement is declared" for exactly that
    reason — the backend can speak to the annotation and must not appear to speak to the
    deployment's governance.

Three ways to close the gap, in increasing order of precision:

1. **An ACL rule carrying `approval: required`** (apcore 0.28.0, PROTOCOL_SPEC §6.1.6) — puts a
   matching call to a human even though the rule's `effect` is `allow`. Because an approval
   requirement also withholds the skill from the public card (FR-AGC-003 criterion 11), this closes
   the discovery exposure and the invocation gap in one move.
2. **`gate_destructive` on the `ExecutionPolicy`** — catches `DELETE`, the only method the toolkit
   marks `destructive`. Catches no `POST`.
3. **A `transform_module` hook** on the scan — the operator's own callable, which can set
   `requires_approval` per operation from anything in the document (an `x-requires-approval` vendor
   extension, a tag, the path). The only option that puts the annotation on the module itself.

### ACL targets and the stability of a derived ID

An ACL `targets` pattern is matched against the `module_id`. On an extensions directory the operator
writes both; on an OpenAPI backend, **the upstream API's authors decide the left-hand side**. A
document that renames `deleteUser` to `removeUser` silently changes the module ID, and a
`targets: ["deleteUser"]` deny rule stops matching — which under `default_effect: allow` is a
fail-open of exactly the shape apcore#112 closed inside the ACL itself. For this binding it is also
a **card** fail-open: the skill reappears on the public card at the same moment it becomes callable.

**Required guidance, and the default the backend should make easy:**

- Set `prefix` and write the catch-all rule against the prefix, not against operation names:

  ```yaml
  apcore-a2a:
    openapi:
      spec: "https://api.example.com/openapi.json"
      prefix: petstore
    acl:
      default_effect: deny
      rules:
        - callers: ["@external"]
          targets: ["petstore.pets.get", "petstore.pets.pet_id.get"]
          effect: allow
          conditions: { identity_types: ["human"] }
        - callers: ["@external"]
          targets: ["petstore.*"]
          effect: deny
  ```

  A prefixed catch-all deny holds no matter what the upstream renames. An allow-list of operation
  names fails **closed** when an ID changes — the safe direction — which is why `default_effect: deny`
  plus a prefixed catch-all is the recommended shape and not merely a stylistic preference. It also
  keeps the public card to exactly the read operations named.
- **`targets: []` is no longer a way to say "everything".** apcore 0.29.0 rejects it at every door.
  Write `["*"]`.
- Static mode (below) removes the moving part entirely: the IDs are frozen in a reviewable file and
  a rename shows up as a diff in a pull request.

---

## Dynamic and static modes

**Dynamic (default).** The spec is fetched and scanned at every process start. Zero redeploy when
the API changes; one fetch and parse per start; a network dependency in the startup path.

**Static.** Scan once at build time into `.binding.yaml` (apcore-toolkit's `YAMLWriter`), load it at
runtime with apcore-toolkit's `BindingLoader` and write it through the same
`HTTPProxyRegistryWriter`. No network dependency at start, no parse cost, and the generated bindings
are reviewable artifacts — an API contract change becomes visible to a human before it ships, and the
module IDs an ACL rule depends on stop moving on their own.

For an internet-facing A2A agent, static mode is the recommended production shape: it removes the
upstream document from the startup path, and it means the set of skills on the public Agent Card
changes only when a human merges a diff.

---

## Configuration

### Config Bus (`apcore-a2a.openapi`)

```yaml
apcore-a2a:
  openapi:
    spec: "https://api.example.com/openapi.json"   # URL or local path — required; path-typed
    base_url: "https://api.example.com"            # defaults to servers[0].url from the document
    prefix: petstore                               # base_path_prefix; required in mixed deployments
    include: "pets.*"                              # scanner include filter
    exclude: "*.internal.*"                        # scanner exclude filter
    include_deprecated: false                      # default true
    acknowledge_unapproved_writes: false           # default false; see FR-OAS-005
    timeout: 30.0                                  # spec fetch timeout, seconds
    headers:                                       # sent with the spec fetch only
      X-Api-Key: "${PETSTORE_SPEC_KEY}"
```

| Key | Default | Notes |
|---|---|---|
| `spec` | — | URL or filesystem path. A URL is taken **verbatim**; a relative path resolves against `Config.project_root`; a set-but-empty value is discarded. See [The spec location is a path-typed key](#the-spec-location-is-a-path-typed-key) |
| `base_url` | `servers[0].url` | Where proxied requests go. Required when the document has no usable absolute `servers[0].url` |
| `prefix` | `null` | Prepended to every derived `module_id`. **Required** when an extensions directory is also configured |
| `include` / `exclude` | `null` | Passed to the scanner's own filters |
| `include_deprecated` | `true` | `false` skips operations marked `deprecated: true` |
| `timeout` | `30.0` | Spec fetch only; not the per-call proxy timeout |
| `headers` | `{}` | Spec fetch only. **Not** forwarded to proxied calls |
| `acknowledge_unapproved_writes` | `false` | Records an explicit operator decision to advertise write-method operations with no approval path on the public card, suppressing FR-OAS-005's warning |

This is the **first nested section** in the `apcore-a2a` namespace, whose five existing keys
(`execution_timeout`, `cors_origins`, `explorer`, `metrics`, `push_notifications`) are all scalars.

### CLI

| Argument | Default | Description |
|---|---|---|
| `--from-openapi` | — | Spec URL or path. Mutually exclusive with `--extensions-dir` unless `--openapi-prefix` is given |
| `--openapi-base-url` | from document | Base URL for proxied requests |
| `--openapi-prefix` | — | `base_path_prefix` for every derived module ID |
| `--openapi-include` / `--openapi-exclude` | — | Scanner filters |
| `--openapi-header` | — | `Key: Value`, repeatable. Spec fetch only |
| `--openapi-no-deprecated` | off | Skip `deprecated: true` operations |

Flag names match apcore-mcp's deliberately: an operator who has configured one bridge should not have
to relearn the surface for the other.

Precedence is the adapter's existing caller-wins rule: an explicit CLI flag or constructor argument
beats the Config Bus, which beats the default.

### The spec location is a path-typed key

`apcore-a2a.openapi.spec` holds either an `http(s)://` URL or a filesystem path, and it is the
**first path-typed key in the `apcore-a2a` namespace**. apcore 0.30.0 declared the closed set of
path-typed configuration keys (PROTOCOL_SPEC §9.2.1) and the base a relative one resolves against
(§9.2.2) — but both apply to apcore's own key surface, and neither extends here.

!!! danger "apcore 0.30.0's path protections do not reach a consumer namespace"
    Measured against apcore 0.30.0, not inferred from the specification:

    - `Config.path_typed_keys()` returns a **hardcoded** set of apcore's own keys
      (`extensions.root`, `extensions.roots[]`, `schema.root`, `acl.root`, `bindings.dir`). It never
      consults a namespace registered through `Config.register_namespace`, even though that method
      accepts a `schema` that could carry `"x-apcore-path": true`.
    - The §9.2.1 requirement-5 empty-value discard is gated on that same fixed set, so
      `APCORE_A2A_OPENAPI_SPEC=` is treated as an ordinary override to `""` — a legal relative path
      to every filesystem API, and never the one an operator meant. This is precisely the failure
      apcore closed for `APCORE_ACL_ROOT=`, in a namespace the fix does not cover.

    The adapter therefore owns these rules rather than inheriting them. The same finding is recorded
    in apcore-mcp's `openapi-backend.md`; both bindings implement it independently because there is
    no shared code path between them.

**Three requirements (FR-OAS-004).**

1. **Discriminate before resolving.** A value beginning `http://` or `https://` is a URL and is used
   verbatim — never path-resolved, never made absolute. Anything else is a filesystem path. The test
   is the scheme prefix, not a guess from the string's shape.
2. **An empty value is not a path.** A set-but-empty `apcore-a2a.openapi.spec` — from
   `APCORE_A2A_OPENAPI_SPEC=` or from `spec: ""` in YAML — MUST be discarded with a WARNING and
   resolution MUST fall through to the next tier, mirroring §9.2.1 requirement 5. It MUST NOT be
   treated as "the working directory".
3. **A relative path resolves against `Config.project_root`.** Not the process CWD, and not the
   OpenAPI document's own directory. `project_root` is apcore 0.30.0's public accessor
   (`Config.project_root` / `projectRoot()` / `Config::project_root()`), carrying no resolution
   behaviour of its own — it reports the base, and the adapter applies it.

!!! success "A new key gets the destination, not the deprecation cycle"
    apcore's §9.2.2 forbids its own four keys from adopting the project-root rule early: they have
    deployed configurations relying on today's two different bases, so §13.2's two-minor deprecation
    window applies.

    `apcore-a2a.openapi.spec` is in the opposite position. It has never shipped, so its deployed
    population is empty and there is nothing to deprecate. The prohibition is on changing keys people
    depend on, not on getting a new one right the first time.

### Upstream credentials

`headers` and `--openapi-header` authenticate the **spec fetch**. They are deliberately not reused
for proxied calls: a document is often public while the API behind it is not, and quietly
broadcasting a spec-read key on every skill invocation would be a privilege escalation the operator
never wrote down.

Credentials for proxied calls arrive through `HTTPProxyRegistryWriter`'s `auth_header_factory` hook,
which is a callable invoked per request and therefore the correct place for a rotating token. It is
reachable from the programmatic API only — there is no CLI flag for it, and there should not be one,
because a static bearer token on a command line is the failure mode the hook exists to avoid.

---

## Security considerations

- **SSRF is the caller's boundary, and the adapter must keep it there.** `load_spec` takes its source
  verbatim and does no allowlisting; the toolkit documents *source* as trusted input. A spec location
  is operator configuration, not caller input, and MUST NOT become reachable from an A2A request —
  there is no "scan this document" skill, and the backend is constructed at startup only.
- **The proxied API inherits the agent's network position.** Every scanned operation executes an
  outbound HTTP call from the server process. An OpenAPI-backed A2A agent placed inside a private
  network turns every allowed skill into a request originating there; the ACL is the control, and
  `default_effect: deny` is the recommended posture for exactly this reason.
- **The public Agent Card is a disclosure surface.** Beyond invocability, the card publishes every
  skill's `description` and input schema. A document describing an internal API discloses its shape
  to any anonymous crawler once scanned. `include` / `exclude` and a prefixed deny rule are the
  controls; FR-OAS-005's warning is the reminder.
- **Spec-fetch headers are secrets.** They MUST NOT be logged, echoed into the Agent Card, or
  included in an error message naming the fetch failure.

---

## Contract: openapi_backend

Builds a `Registry` from an OpenAPI document.

### Inputs

| Parameter | Type | Required | Description |
|---|---|---|---|
| `spec` | `str` / `string` / `&str` | yes | URL or filesystem path, resolved per FR-OAS-004 |
| `base_url` | `str \| None` | no | Overrides `servers[0].url` |
| `prefix` | `str \| None` | no | `base_path_prefix` for every derived module ID |
| `include` / `exclude` | `str \| None` | no | Scanner filters |
| `include_deprecated` | `bool` | no | Default `true` |
| `timeout` | `float` | no | Spec fetch timeout, default `30.0` |
| `headers` | `dict \| None` | no | Spec fetch only |
| `transform_module` | callable \| `None` | no | Per-module hook, applied before registration |
| `auth_header_factory` | callable \| `None` | no | Passed to `HTTPProxyRegistryWriter` |

### Errors

| Condition | Error |
|---|---|
| `prefix` absent while an extensions directory is also configured | `ValueError` naming `prefix`. Checked **first**, before any fetch or scan |
| `spec` empty after FR-OAS-004 discard, with no fallback tier | `ValueError` / `Error` naming the key |
| Spec fetch or parse failure | propagated from `load_spec`, naming the resolved location, **never the headers** |
| Document is not OpenAPI 3.0.x / 3.1.x | propagated from `OpenAPIScanner.scan` |
| No usable `base_url` | `ValueError` naming both the document and the config key |
| Derived `module_id` collides with a module already in the registry | `ValueError` naming **every** colliding ID, sorted and deduplicated, before anything is written — reporting only the first forces one restart per collision |

Two failures are deliberately **not** errors: a module whose ID cannot be projected is dropped with a
WARNING (FR-OAS-002), and a per-module write failure is logged at ERROR while the remaining modules
still register. Neither leaves the registry partially populated with respect to the *set* the
preflight approved.

### Returns

A populated `Registry`. Never a partially-populated one: the collision preflight and every
validation run before the first `write`.

### Properties

- **Idempotent** for a fixed document: the same spec yields the same module IDs, byte-identical
  across all three SDKs (pinned by apcore-toolkit's own corpus plus this repository's
  `projection_*` cases).
- **Side effects:** one outbound HTTP GET when `spec` is a URL; no filesystem writes in dynamic mode.
- **Emits**, in this order, before returning: the FR-OAS-002 projection-drop WARNINGs, any per-module
  scanner warnings, a zero-modules-produced WARNING, the FR-OAS-003 description-fallback INFO line,
  per-module write-failure ERRORs (non-fatal), and the FR-OAS-005 unapproved-write WARNING.
- **Every registered module ID is apcore-legal**, and every registered module carries a non-empty
  description. These two invariants are what FR-OAS-002 and FR-OAS-003 exist to hold, and they are
  the reason the backend cannot be a thin `scan`-then-`write` call.

---

## Canonical diagnostic text

The six report sites emit at levels that are already normative and identical across the three
SDKs. Their **text** drifted, and a cross-language audit found five of the six carrying
different facts rather than different phrasing — Python's projection-drop message named no
remedy, its collision error did not say the registry was untouched, and its no-`base_url` error
did not say what breaks. Each of those is a fact the operator needs and one binding withheld.

The table fixes what each message must **contain**. Wording may differ where a language must
name its own API; the listed facts may not go missing.

| Site | Level | MUST contain |
|---|---|---|
| Projection drop (FR-OAS-002) | `warn` | the derived ID; the offending segment, separately identifiable from the ID; the required segment pattern `^[a-z][a-z0-9_]*$`; and the remedy — supply a `derive_module_id` / `transform_module` hook (spelled in the host language) |
| Per-module scanner warning | `warn` | the module ID and the scanner's own text, unmodified |
| Zero modules produced | `warn` | that the Agent Card will have no skills |
| Description synthesis (FR-OAS-003) | `info` | the count, the denominator, and every affected ID **post-projection** |
| Per-module write failure | `error` | the module ID and the writer's `verification_error` |
| Unapproved writes (FR-OAS-005) | `warn` | the count; the methods; the public Agent Card **and its path**; the three remedies |

And for the three fatal errors:

| Error | MUST contain |
|---|---|
| ID collision (FR-OAS-006) | every colliding ID, sorted and deduplicated; **"Nothing was registered."**; and the `prefix` remedy |
| No `base_url` | that the document has no usable absolute `servers[0].url`, **that every proxied call would otherwise resolve against an unknown host**, and the config key to set |
| Missing `prefix` in a mixed deployment | the key `prefix`, and why — a derived ID can collide with a project module ID |

!!! note "Why the required-pattern fragment is in the drop message"
    An operator reading *"the derived module ID has a segment (`2fa`) apcore's registry cannot
    accept"* has been told what failed and not what would succeed. `2fa` is rejected because a
    segment may not begin with a digit — which is not guessable from the message without the
    pattern. Rust carried the fragment, the other two did not; it is now required of all three.

---

## Conformance

Shared fixture: `conformance/fixtures/openapi_backend.json`. It asserts the parts this binding owns —
the scanner's own behaviour is pinned by apcore-toolkit's corpus and is not restated.

| Case group | Asserts |
|---|---|
| `projection_*` | FR-OAS-002: `listPets` → `listpets`; `pet-store.items.get` → `pet_store.items.get`; an unprojectable segment (`2fa`) dropped **with a warning naming the ID and the segment**; `listPets` + `listpets` deduplicated to `listpets` / `listpets_2` (pins projection-before-dedup) |
| `spec_location_*` | FR-OAS-004's three rules: URL verbatim, empty discarded with fallthrough, relative resolved against `project_root` **and not CWD** — the case supplies a differing `cwd`, which is what makes it assert anything |
| `description_fallback_*` | FR-OAS-003: synthesized `{METHOD} {path}` for empty/whitespace descriptions; untouched when a description exists; the operation reaches the Agent Card either way |
| `write_warning_*` | FR-OAS-005: fires for each of POST/PUT/PATCH/DELETE; silent for a read-only document; silent under `acknowledge_unapproved_writes`; **still fires under a permissive ACL** |
| `collision_*` | A derived-vs-project collision fails at startup naming **every** colliding ID, and leaves the registry byte-for-byte unchanged |
| `public_card_*` | A scanned write operation with no approval requirement **is** on the public card; the same operation under an ACL rule carrying `approval: required` is **not**, and is on the extended card |

Two discriminating cases carry most of the weight:

- `unprojectable_segment_skipped_with_warning` — an implementation that merely drops the module
  passes every other assertion in the `projection_*` group and fails this one.
- `write_warning_not_suppressed_by_permissive_acl` — an implementation that gates the warning on
  `governance_state().acl_configured` passes every other case in its group and fails this one.

Expectations MUST be computed by running the real apcore-toolkit scanner, never derived by reading
its algorithm — that is how apcore-mcp's corpus was built, and the projection cases in particular
depend on behaviour (`deduplicate_ids` ordering) that the prose does not fully determine.

---

## Divergences deliberately not inherited from apcore-mcp

apcore-mcp shipped this feature first, and a line-by-line comparison of its three implementations
found four defects. They are recorded here because the obvious way to build this feature is to port
that code, and porting it faithfully would reproduce all four.

| Defect in apcore-mcp | Upstream | What apcore-a2a MUST do |
|---|---|---|
| **Rust drops `headers` entirely.** No field on the options struct, and `cli.rs` builds a header map from `--openapi-header` then never passes it — the flag is a silent no-op | [apcore-mcp-rust#8](https://github.com/aiperceivable/apcore-mcp-rust/issues/8) | Thread `headers` through to `load_spec_with_options` in all three SDKs, or do not offer the flag |
| **TypeScript passes seconds where the toolkit wants milliseconds.** `LoadSpecOptions.timeout` is in ms (default `30_000`); the config default `30` is passed straight through, making a documented `timeout: 30.0` a **30 ms** fetch timeout | [apcore-mcp-typescript#10](https://github.com/aiperceivable/apcore-mcp-typescript/issues/10) | Convert at the boundary. The Config Bus key is seconds in every SDK; the TS call site multiplies by 1000 |
| **`timeout` configures two different things.** Python/TS apply it to the spec fetch (spec-conformant); Rust applies it to the *writer*, i.e. the per-call proxy timeout, and never to the fetch | [apcore-mcp-rust#9](https://github.com/aiperceivable/apcore-mcp-rust/issues/9) | `timeout` is the spec-fetch timeout in all three, per the table above. A proxy timeout, if ever needed, gets its own key |
| **The Config-Bus route ignores `project_root` in TS and Rust.** Neither `build_*_from_config` sets it, so a relative `spec` silently resolves against the CWD — contradicting FR-OAS-004 rule 3 on the one route most deployments use | [apcore-mcp#19](https://github.com/aiperceivable/apcore-mcp/issues/19) | Resolve `project_root` from `Config` inside the config route in all three SDKs, as apcore-mcp's Python does |

These are reported upstream rather than only worked around here; the entries stay until apcore-mcp
closes them, so a future reader can tell a deliberate difference from drift. Two of the four are
violations of apcore-mcp's own specification rather than of a preference of this binding: its
`openapi-backend.md` §Configuration documents `timeout` as "Spec fetch only; not the per-call proxy
timeout" and `--openapi-header` as "Spec fetch only".

---

## See Also

- apcore-toolkit `openapi-scanner.md` — the scanner, the writer, and the `module_id` derivation
- [Adapters](adapters.md) — `SkillMapper`, `AgentCardBuilder`, and the card-visibility filters
- [Public API](public-api.md) — the backend-source entry points this feature extends
- [CLI](cli.md) — the flag surface
- apcore-mcp `openapi-backend.md` — the sibling binding's version of this feature
