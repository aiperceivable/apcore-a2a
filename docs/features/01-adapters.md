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
        registry: object,        # duck-typed: has list() + get_definition()
        *,
        name: str,
        description: str,
        version: str,
        url: str,
        capabilities: AgentCapabilities,
        security_schemes: dict | None = None,
    ) -> AgentCard:
        """Returns a2a.types.AgentCard Pydantic model."""

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
5. Compute `capabilities`:
   - `streaming`: True if executor has `stream()` method.
   - `pushNotifications`: from configuration.
   - `stateTransitionHistory`: from task store config.
6. Build card dict:
   ```json
   {
     "name": "...", "description": "...", "version": "...", "url": "...",
     "skills": [...],
     "capabilities": {"streaming": bool, "pushNotifications": bool, "stateTransitionHistory": bool},
     "defaultInputModes": ["text/plain", "application/json"],
     "defaultOutputModes": ["application/json"]
   }
   ```
7. Cache in `self._cached_card`. Extended card in `self._cached_extended_card`.

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
    def _build_extensions(self, annotations: object | None) -> dict | None: ...
```

**Field mapping:**

| apcore field | A2A Skill field | Notes |
|---|---|---|
| `module_id` | `id` | Verbatim |
| humanize(`module_id`) | `name` | Dots/underscores → spaces, title case |
| `description` | `description` | Verbatim. Empty/None → return `None` |
| `tags` | `tags` | Verbatim |
| `examples[:10]` | `examples` | `title` → `name`, `inputs` → JSON string in TextPart |
| computed | `inputModes` | See mode logic below |
| computed | `outputModes` | See mode logic below |
| `annotations` | `extensions.apcore.annotations` | All 5 boolean flags |

**Input/output mode logic:**

| Condition | inputModes | outputModes |
|---|---|---|
| `input_schema` with root type `object` | `["application/json"]` | |
| `input_schema` with root type `string` | `["application/json", "text/plain"]` | |
| No `input_schema` | `["text/plain"]` | |
| `output_schema` defined | | `["application/json"]` |
| No `output_schema` | | `["text/plain"]` |

**Extensions structure (when annotations is not None):**
```json
{
  "apcore": {
    "annotations": {
      "readonly": false, "destructive": false, "idempotent": false,
      "requires_approval": false, "open_world": true
    }
  }
}
```

---

### 3. `SchemaConverter` — `adapters/schema.py`

Converts apcore JSON Schemas for A2A DataPart usage. Reuses logic from apcore-mcp's SchemaConverter.

```python
class SchemaConverter:
    MAX_DEPTH = 32

    def convert_input_schema(self, descriptor: object) -> dict:
        """Convert input_schema: inline $refs, strip $defs, ensure root type=object."""

    def convert_output_schema(self, descriptor: object) -> dict:
        """Convert output_schema for Artifact metadata annotation."""

    def detect_root_type(self, schema: dict | None) -> str:
        """Return 'string', 'object', or 'unknown'."""

    def _inline_refs(self, schema: dict, defs: dict, depth: int, visited: set) -> dict:
        """Recursively resolve $ref. Raises ValueError on circular refs or depth > 32."""
```

**Conversion rules:**
- `None` or `{}` → `{"type": "object", "properties": {}}`
- Deep-copy before mutation (never mutate input)
- Resolve all `#/$defs/...` references inline
- Remove `$defs` key from result
- Max recursion depth: 32 (raise `ValueError("Schema $ref depth limit exceeded")`)
- Circular ref detected via `visited: set[str]` → raise `ValueError("Circular $ref detected")`

---

### 4. `ErrorMapper` — `adapters/errors.py`

Maps apcore exceptions to A2A JSON-RPC error dicts with security-aware sanitization.

```python
class ErrorMapper:
    def to_jsonrpc_error(self, error: Exception) -> dict:
        """Returns: {"code": int, "message": str, "data": dict | None}"""

    def _build_validation_data(self, error: Exception) -> dict:
        """For SchemaValidationError: extract field-level detail."""

    def _sanitize_message(self, message: str) -> str:
        """Strip paths, tracebacks. Truncate to 500 chars."""
```

**Error map:**

| apcore Exception | code | message | sanitized |
|---|---|---|---|
| `ModuleNotFoundError` | -32601 | `"Skill not found: {module_id}"` | No |
| `SchemaValidationError` | -32602 | `"Invalid params"` | No (field details in `data.errors`) |
| `ACLDeniedError` | -32001 | `"Task not found"` | **Yes** (masks real type) |
| `ModuleExecuteError` | -32603 | `"Internal error"` | Yes |
| `ModuleTimeoutError` | -32603 | `"Execution timed out"` | Yes |
| `InvalidInputError` | -32602 | `"Invalid input: {description}"` | No |
| `CallDepthExceededError` | -32603 | `"Safety limit exceeded"` | Yes |
| `CircularCallError` | -32603 | `"Safety limit exceeded"` | Yes |
| `CallFrequencyExceededError` | -32603 | `"Safety limit exceeded"` | Yes |
| `ApprovalPendingError` | N/A | Task transitions to `input_required` | N/A |
| Any other `Exception` | -32603 | `"Internal error"` | Yes |

**Sanitization rules:**
1. Strip substrings matching file path pattern `r'/[^\s]+/[^\s]+'`.
2. Strip traceback lines (`Traceback`, `File "`, `line \d+`).
3. Truncate to 500 characters.
4. Log full unsanitized exception at ERROR level with stack trace.

---

### 5. `PartConverter` — `adapters/parts.py`

Bidirectional converter between A2A Parts and apcore module inputs/outputs.

```python
class PartConverter:
    def __init__(self, schema_converter: SchemaConverter) -> None: ...

    def parts_to_input(self, parts: list[dict], descriptor: object) -> dict | str:
        """Convert A2A message Parts to apcore module input.

        Rules:
        1. Empty parts → raise ValueError("Message must contain at least one Part")
        2. First DataPart(mediaType="application/json") → return data dict
        3. TextPart + input_schema root type 'string' → return text string
        4. TextPart + input_schema root type 'object' → JSON.parse(text)
           - parse failure → raise ValueError("Invalid JSON in TextPart")
        5. FilePart → return {"uri": ..., "name": ..., "mimeType": ...}
        """

    def output_to_parts(self, output: object) -> a2a.types.Artifact:
        """Convert apcore module output to an a2a.types.Artifact Pydantic model.

        Rules:
        1. None → []
        2. dict → [DataPart(data=output, mediaType="application/json")]
        3. str → [TextPart(text=output)]
        4. bytes → [FilePart(bytes=base64(output), mimeType=detected)]
        """
```

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
