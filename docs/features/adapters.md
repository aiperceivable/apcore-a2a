# Feature: Adapters Module

| Field | Value |
|-------|-------|
| Feature ID | F-01 |
| Name | adapters |
| Priority | P0 |
| SRS Refs | FR-AGC-001..005, FR-SKL-001..004, FR-MSG-003, FR-MSG-004, FR-ERR-001..008 |
| Tech Design | §4.3 Adapters Module |
| Depends On | None (foundation layer) |
| Blocks | F-03 (server-core), F-08 (public-api) |

## Purpose

Five pure-logic converter classes that transform apcore module metadata and execution results into A2A-compatible structures. No I/O, no network, no side effects — all deterministic, easily unit-testable.

## Components

### 1. `AgentCardBuilder` — `adapters/agent_card.py`

Builds A2A Agent Card dicts from Registry metadata.

```python
class AgentCardBuilder:
    def __init__(self, skill_mapper: SkillMapper) -> None: ...

    def build(
        self,
        registry: Any,           # duck-typed: has list() + get_definition()
        *,
        name: str,
        description: str,
        version: str,
        url: str,
        capabilities: AgentCapabilities,
        security_schemes: Any | None = None,
    ) -> AgentCard:
        """Returns a2a.types.AgentCard Pydantic model."""

    def get_cached_or_build(
        self,
        registry: Any,
        *,
        name: str,
        description: str,
        version: str,
        url: str,
        capabilities: AgentCapabilities,
        security_schemes: Any | None = None,
    ) -> AgentCard:
        """Return cached card if available, otherwise build a new one."""

    def build_extended(
        self,
        *,
        base_card: AgentCard,
    ) -> AgentCard:
        """Extended card: deep copy of base_card. Override in subclass to add restricted skills."""

    def invalidate_cache(self) -> None:
        """Called by RegistryListener on module add/remove."""
```

**Build logic:**
1. Call `registry.list()` → module IDs.
2. For each: `registry.get_definition(module_id)` → `ModuleDescriptor`.
3. Skip modules with `description == ""` or `None` (log warning: `"Skipping module {module_id}: missing description"`).
4. Convert each to Skill via `SkillMapper.to_skill(descriptor)`.
5. Compute `capabilities` (A2A 1.0 `AgentCapabilities`):
   - `streaming`: True (executor streaming is always available; non-streaming modules fall back to a single chunk).
   - `pushNotifications`: from configuration.
   - `extensions`: list of A2A protocol extensions (`[]` by default).
   - `extendedAgentCard`: True when security schemes are configured (replaces 0.3's top-level `supportsAuthenticatedExtendedCard`).
6. Build the **A2A 1.0** card. The 0.3 top-level `url` and `protocolVersion` are
   replaced by `supportedInterfaces`; security requirements and signatures are
   first-class:
   ```json
   {
     "name": "...", "description": "...", "version": "...",
     "supportedInterfaces": [
       {"url": "...", "protocolBinding": "JSONRPC", "protocolVersion": "1.0", "tenant": ""}
     ],
     "provider": null,
     "skills": [...],
     "capabilities": {"streaming": true, "pushNotifications": false, "extensions": [], "extendedAgentCard": false},
     "defaultInputModes": ["text/plain", "application/json"],
     "defaultOutputModes": ["text/plain", "application/json"],
     "securitySchemes": {},
     "securityRequirements": [],
     "signatures": []
   }
   ```
7. Cache in `self._cached_card`. Extended card in `self._cached_extended_card`.

> The card is served at `/.well-known/agent-card.json` (A2A 1.0). A
> `/.well-known/agent.json` alias is also served for 0.3 client compatibility.

---

### 2. `SkillMapper` — `adapters/skill_mapper.py`

Converts `ModuleDescriptor` to A2A Skill dict.

```python
class SkillMapper:
    def to_skill(self, descriptor: object) -> AgentSkill | None:
        """Returns a2a.types.AgentSkill or None if module has no description."""

    def _humanize_module_id(self, module_id: str) -> str:
        # "image.resize" -> "Image Resize"
        return module_id.replace(".", " ").replace("_", " ").title()

    def _compute_input_modes(self, descriptor: object) -> list[str]: ...
    def _compute_output_modes(self, descriptor: object) -> list[str]: ...
    def _build_examples(self, descriptor: object) -> list[str]: ...  # max 10, title strings only
    # Note: _build_extensions() was removed — AgentSkill has no `extensions` field in a2a-sdk
```

**Field mapping:**

| apcore field | A2A Skill field | Notes |
|---|---|---|
| `module_id` | `id` | Always uses module_id |
| `metadata["display"]["a2a"]["alias"]` or `metadata["display"]["alias"]` or humanized `module_id` | `name` | Display overlay alias takes priority |
| `metadata["display"]["a2a"]["description"]` or `metadata["display"]["description"]` or `description` | `description` | Display overlay description takes priority. Guidance appended if present. Empty/None → return `None` |
| `metadata["display"]["tags"]` or `tags` | `tags` | Display overlay tags take priority |
| `examples[:10]` | `examples` | `title` → `name`, `inputs` → JSON string in TextPart |
| computed | `inputModes` | See mode logic below |
| computed | `outputModes` | See mode logic below |
| `[]` | `securityRequirements` | Required by A2A 1.0 `AgentSkill`; empty by default |
| `annotations` | *(not mapped)* | See note below |

**Input/output mode logic:**

| Condition | inputModes | outputModes |
|---|---|---|
| `input_schema` with root type `object` | `["application/json"]` | |
| `input_schema` with root type `string` | `["application/json", "text/plain"]` | |
| No `input_schema` | `["text/plain"]` | |
| `output_schema` defined | | `["application/json"]` |
| No `output_schema` | | `["text/plain"]` |

**Note on annotations:** `_build_extensions()` has been removed. `a2a.types.AgentSkill` has no `extensions` field in the A2A SDK; apcore annotations are available via the Explorer UI's `_inputSchemas` enrichment instead.

---

### 3. `SchemaConverter` — `adapters/schema.py`

Converts apcore JSON Schemas for A2A DataPart usage. `$ref` resolution is
delegated to the shared **apcore-toolkit** resolver (`deep_resolve_refs` /
`deepResolveRefs`), the same helper used by apcore-mcp and apcore-cli.

```python
from apcore_toolkit import deep_resolve_refs

class SchemaConverter:
    def convert_input_schema(self, descriptor: object) -> dict:
        """Convert input_schema: inline $refs, strip $defs, ensure root type=object."""

    def convert_output_schema(self, descriptor: object) -> dict:
        """Convert output_schema for Artifact metadata annotation."""

    def detect_root_type(self, schema: dict | None) -> str:
        """Return 'string', 'object', or 'unknown'."""

    def _convert_schema(self, schema: dict | None) -> dict:
        """Deep-copy, delegate $ref inlining to deep_resolve_refs(schema, schema),
        strip $defs, then ensure root type=object."""

    def _ensure_object_type(self, schema: dict) -> dict:
        """Ensure root schema has type: object."""
```

**Conversion rules:**
- `None` or `{}` → `{"type": "object", "properties": {}}`
- Deep-copy before mutation (never mutate input)
- Resolve all `#/$defs/...` references inline via `deep_resolve_refs`
- Remove `$defs` key from result
- The toolkit resolver is **depth-capped (16) and non-throwing**: circular,
  missing, or non-pointer `$ref`s resolve to a partial schema or `{}` rather
  than raising (shared cross-SDK behavior, replacing the prior hand-rolled
  resolver that raised `ValueError` on cycles/depth).

---

### 4. `ErrorMapper` — `adapters/errors.py`

Maps apcore exceptions to A2A JSON-RPC error dicts with security-aware sanitization.

```python
class ErrorMapper:
    def to_jsonrpc_error(self, error: Exception) -> dict:
        """Returns: {"code": int, "message": str}"""

    def _handle_apcore_error(self, error: Exception, error_code: str) -> dict:
        """Handle apcore errors with a .code attribute (string matching)."""

    def _sanitize_message(self, message: str) -> str:
        """Strip paths, tracebacks. Truncate to 500 chars."""
```

**Error dispatch:** Errors are matched by string `.code` attribute (e.g., `"MODULE_NOT_FOUND"`),
not by exception class names.

**Error map:**

| apcore `.code` | JSON-RPC code | message | sanitized |
|---|---|---|---|
| `MODULE_NOT_FOUND` | -32601 | sanitized original message | Yes |
| `SCHEMA_VALIDATION_ERROR` | -32602 | sanitized original message | Yes |
| `ACL_DENIED` | -32001 | `"Task not found"` | **Yes** (masks real type) |
| `MODULE_TIMEOUT` / `EXECUTION_TIMEOUT` | -32603 | `"Execution timeout"` | No |
| `INVALID_INPUT` | -32602 | `"Invalid input: {sanitized description}"` | Yes |
| `CALL_DEPTH_EXCEEDED` / `CIRCULAR_CALL` / `CALL_FREQUENCY_EXCEEDED` | -32603 | `"Safety limit exceeded"` | No |
| `MODULE_DISABLED` | -32603 | `"Module is currently disabled"` | No |
| `CONFIG_NAMESPACE_DUPLICATE` / `CONFIG_MOUNT_ERROR` / `CONFIG_BIND_ERROR` | -32603 | `"Configuration error"` | No |
| `asyncio.TimeoutError` | -32603 | `"Execution timeout"` | No |
| Any other `Exception` | -32603 | `"Internal server error"` | No |

**Sanitization rules:**
1. Strip substrings matching file path pattern `r'~?/[^\s]*'` (Unix paths and `~` paths).
2. Strip traceback lines (`Traceback`, `File "`, `line \d+`).
3. Truncate to 500 characters.
4. Log full unsanitized exception at ERROR level with stack trace.

---

### 5. `PartConverter` — `adapters/parts.py`

Bidirectional converter between A2A Parts and apcore module inputs/outputs.

```python
class PartConverter:
    def __init__(self, schema_converter: SchemaConverter | None = None) -> None:
        """schema_converter defaults to SchemaConverter() if not provided."""

    def parts_to_input(self, parts: list[Part], descriptor: Any) -> dict | str:
        """Convert A2A message Parts to apcore module input.

        Rules:
        1. Empty parts → raise ValueError("Message must contain at least one Part")
        2. Multiple parts → raise ValueError("Multiple parts are not supported; expected exactly one Part")
        3. DataPart → return data dict
        4. TextPart + input_schema root type 'string' → return text string
        5. TextPart + input_schema root type 'object' → JSON.parse(text)
           - parse failure → raise ValueError("TextPart text is not valid JSON: {error}")
        6. FilePart → raise ValueError("FilePart is not supported")
        """

    def output_to_parts(self, output: Any, task_id: str = "") -> Artifact:
        """Convert apcore module output to an a2a.types.Artifact Pydantic model.

        Rules:
        1. None → Artifact(artifact_id=..., parts=[])
        2. dict → Artifact with [DataPart(data=output)]
        3. str → Artifact with [TextPart(text=output)]
        4. list → Artifact with [TextPart(text=json.dumps(output))]
        5. Other → Artifact with [TextPart(text=str(output))]
        """
```

> **A2A 1.0 Part shape.** `Part` is a protobuf `oneof`: a text part carries
> `content = text`, a data part carries `content = data`. On the wire it
> serializes flat — `{"text": "..."}` or `{"data": {...}}` — with no `type`
> discriminator. Reading inspects the active oneof case (Python:
> `part.WhichOneof("content")`; TypeScript: `part.content.$case`; Rust: the
> `Part` enum variant). A "FilePart" corresponds to the `raw` (bytes) or `url`
> oneof case and is rejected as unsupported.

## File Structure

```
src/apcore_a2a/adapters/
    __init__.py          # exports: AgentCardBuilder, SkillMapper, SchemaConverter,
                         #          ErrorMapper, PartConverter
    agent_card.py        # AgentCardBuilder
    skill_mapper.py      # SkillMapper
    schema.py            # SchemaConverter
    errors.py            # ErrorMapper
    parts.py             # PartConverter
```

## Key Invariants

- No I/O (no network, no file system, no subprocess)
- All methods are pure or nearly-pure (deterministic given inputs)
- Deep-copy all input schemas before mutation
- `to_skill()` returns `None` for invalid modules (never raises)
- Error sanitization: never expose caller identity, file paths, or stack traces to callers

## Test Module

`tests/adapters/` — all unit tests, no fixtures beyond stubs
