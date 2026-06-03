# Conformance Suite

## Purpose

This directory contains cross-language parity test fixtures that all three SDK
implementations — **Python** (`apcore-a2a-python`), **TypeScript**
(`apcore-a2a-typescript`), and **Rust** (`apcore-a2a-rust`) — must pass. The
fixtures define the canonical, observable behavior of an apcore-a2a server and
client on the A2A 1.0 wire protocol.

The goal is to guarantee that an agent (or a human) switching between SDK
implementations gets identical observable behavior: the same Agent Card shape,
the same JSON-RPC error codes and messages, the same SSE event sequence, and
the same `Identity` derived from a given JWT.

These fixtures are the executable counterpart of the cross-language decisions
recorded in `docs/features/*.md` and the test cases in
`docs/spec/test-plan.md`. Where the test plan describes a behavior in prose per
language, the conformance fixtures pin the exact bytes/values every language
must agree on.

---

## Algorithm A01

**Algorithm A01** is the A2A wire-format and behavior parity algorithm. It is
the normative specification that all SDK implementations must conform to.

The algorithm states:

> For any given input (JWT, module descriptor, JSON-RPC request, or module
> output), every conforming apcore-a2a SDK MUST produce the same A2A 1.0 wire
> output: identical JSON-RPC error codes and sanitized messages, identical
> Agent Card field shapes, an identical SSE event sequence, and an identical
> derived `Identity` (or identical rejection).

Concretely, A01 is decomposed into the fixture files below. Each file carries
its own `description` and `contract_version` and is consumed independently by
every SDK's conformance runner.

| Fixture | Sub-algorithm | Locks |
|---|---|---|
| `fixtures/jwt_claim_coercion.json` | A-AUTH | JWT claim → `Identity` coercion (Rust-strict canonical rule) |
| `fixtures/agent_card.json` | A-CARD | Agent Card field shapes, incl. `securitySchemes` proto3 `oneof` form |
| `fixtures/error_mapping.json` | A-ERR | apcore exception → JSON-RPC error code + sanitized message |
| `fixtures/skill_resolution.json` | A-SKILL | missing/invalid `skillId` and unparseable parts → FAILED task (not `-32602`) |
| `fixtures/streaming_events.json` | A-STREAM | SSE event sequence incl. terminal `lastChunk` empty-artifact marker |
| `fixtures/part_conversion.json` | A-PART | A2A `Part` ⇄ module input/output conversion |

### JSON-RPC Error Code Table

| Code | A2A meaning | apcore source exception |
|---|---|---|
| `-32700` | Parse error | malformed JSON body |
| `-32600` | Invalid Request | missing/invalid `jsonrpc` field, or missing `message` envelope |
| `-32601` | Method not found | unknown JSON-RPC method, `ModuleNotFoundError` |
| `-32602` | Invalid params | `SchemaValidationError`, missing `message` envelope |
| `-32001` | Task not found | unknown task id; **also** `ACLDeniedError` (detail-suppressed to prevent disclosure) |
| `-32002` | Task not cancelable | `tasks/cancel` on a task in a terminal state |
| `-32003` | Push notifications not supported | config method when server started with `push_notifications=False` |
| `-32603` | Internal error | `ModuleExecuteError` and any unrecognized exception → fixed `"Internal server error"` ("Internal error" is the JSON-RPC *name* of code -32603, not the emitted message) |

> **Sanitization invariant (NFR-SEC-003).** For `-32001` (ACL) and `-32603`
> (internal) errors, the message MUST NOT contain caller identities, module
> names, stack traces, file paths, internal variable names, or configuration
> values. The conformance runner asserts the *absence* of the sensitive
> substrings supplied in each error case.

---

## Fixture Format

Every fixture is a single JSON object with this top-level shape:

```json
{
  "description": "<one-line statement of what this fixture locks>",
  "contract_version": "1.0",
  "sub_algorithm": "A-XXX",
  "test_cases": [ ... ],
  "error_cases": [ ... ],
  "notes": { "known_divergences": [ ... ] }
}
```

- `test_cases` — happy-path cases. Each has a unique `id`, an `input` block,
  and an `expected_*` block. Assertions are **partial matches**: the actual
  output MUST contain every key/value in the expected block; extra keys are
  allowed.
- `error_cases` — cases that must be rejected. Each has an `id`, an `input`,
  and either `expected_error_code` (JSON-RPC integer) and/or
  `expected_error_message` / `expected_message_excludes` (substring
  assertions).
- `notes.known_divergences` — documented, *accepted* differences between SDKs
  that the assertions deliberately do **not** check (e.g. internal field types
  that never reach the wire). Anything listed here is out of scope for parity.

Field-level conventions follow the rest of the ecosystem
(`apcore-cli/conformance`, `apcore-mcp/conformance`):

- Fixture keys are **snake_case**. SDKs in camelCase languages (TypeScript) map
  them at the test-loader boundary.
- `string_uuid_v4` is a sentinel meaning "assert this field matches the UUID v4
  regex", not a literal match.
- A2A 1.0 enum values are spelled in full (`TASK_STATE_COMPLETED`,
  `ROLE_AGENT`), never the 0.3 short forms.

---

## How to Use

Each SDK ships a conformance runner that loads these fixtures and drives them
through its own public API (not a subprocess — apcore-a2a is a library, so the
runner constructs the adapter/server in-process).

1. **Load the fixture file** for the sub-algorithm under test.
2. **For `test_cases`:** feed `input` to the relevant entry point
   (`JWTAuthenticator.authenticate`, `AgentCardBuilder.build`,
   `ErrorMapper.to_jsonrpc_error`, the JSON-RPC dispatcher, `PartConverter`),
   then partial-match the result against `expected_*`.
3. **For `error_cases`:** assert the produced JSON-RPC error `code` equals
   `expected_error_code`, that the message contains every
   `expected_error_message` substring, and that it contains **none** of the
   `expected_message_excludes` substrings.
4. **Report by `id`.** Failure messages must name the fixture file and case
   `id` so CI reports point at the exact divergence.

### Example (Python — A-ERR)

```python
import json
from apcore_a2a.adapters.error_mapper import ErrorMapper
from apcore import ModuleExecuteError, ACLDeniedError

cases = json.load(open("conformance/fixtures/error_mapping.json"))

for case in cases["error_cases"]:
    err = build_exception(case["input"])          # reconstruct the apcore exception
    rpc = ErrorMapper.to_jsonrpc_error(err)
    assert rpc.code == case["expected_error_code"], f"[{case['id']}] code"
    for needle in case.get("expected_error_message", []):
        assert needle in rpc.message, f"[{case['id']}] missing {needle!r}"
    for forbidden in case.get("expected_message_excludes", []):
        assert forbidden not in rpc.message, f"[{case['id']}] leaked {forbidden!r}"
```

---

## Contributing

To add or change a parity behavior:

1. Decide the **single** canonical behavior first and record the rationale in
   the relevant `docs/features/*.md` file (e.g. the claim-coercion table in
   `auth.md`). The fixture encodes that decision; it does not invent it.
2. Add a case to the appropriate `fixtures/*.json` with a unique, descriptive
   snake_case `id`. Include only the assertion fields meaningful to the case.
3. If a behavior genuinely cannot be unified across all three languages,
   document it under `notes.known_divergences` and ensure the assertions skip
   it — do **not** silently weaken a check.
4. Run the conformance suite against Python, TypeScript, and Rust. All three
   must pass before merging.
5. Do not rename existing case `id` values — CI pipelines reference them.

> Parity decisions already locked (2026-06-02 sync): JWT coercion is
> Rust-strict; `securitySchemes` is served in proto3 `oneof` shape while
> `securityRequirements` is always `[]`; Rust emits the terminal `lastChunk`
> empty-artifact marker; missing/invalid `skillId` yields a FAILED task event,
> not a `-32602`. These fixtures exist to keep those decisions from regressing.
</content>
