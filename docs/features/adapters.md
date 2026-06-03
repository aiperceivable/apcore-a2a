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

=== "Python"

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

=== "TypeScript"

    ```typescript
    // src/adapters/agent-card.ts
    class AgentCardBuilder {
      constructor(skillMapper: SkillMapper)

      build(
        registry: unknown,
        opts: {
          name: string;
          description: string;
          version: string;
          url: string;
          capabilities: AgentCapabilities;
          securitySchemes?: Record<string, unknown>;
        },
      ): AgentCard

      getCachedOrBuild(
        registry: unknown,
        opts: {
          name: string;
          description: string;
          version: string;
          url: string;
          capabilities: AgentCapabilities;
          securitySchemes?: Record<string, unknown>;
        },
      ): AgentCard

      buildExtended(baseCard: AgentCard): AgentCard

      invalidateCache(): void
    }
    ```

=== "Rust"

    ```rust
    // src/adapters/agent_card.rs
    impl AgentCardBuilder {
        pub fn new(skill_mapper: SkillMapper) -> Self

        pub fn build(
            &self,
            registry: &Registry,
            name: &str,
            description: &str,
            version: &str,
            url: &str,
            capabilities: AgentCapabilities,
            security_schemes: Option<Value>,
        ) -> AgentCard

        pub fn get_cached_or_build(
            &self,
            registry: &Registry,
            name: &str,
            description: &str,
            version: &str,
            url: &str,
            capabilities: AgentCapabilities,
            security_schemes: Option<Value>,
        ) -> AgentCard

        pub fn build_extended(&self, base_card: &AgentCard) -> AgentCard

        pub fn invalidate_cache(&self)
    }
    ```

**Build logic:**
1. Call `registry.list()` → module IDs.
2. For each: `registry.get_definition(module_id)` → `ModuleDescriptor`.
3. Skip modules with an empty, `None`, or whitespace-only `description` (the description is trimmed before the check; log warning: `"Skipping module {module_id}: missing description"`).
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

=== "Python"

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

=== "TypeScript"

    ```typescript
    // src/adapters/skill-mapper.ts
    class SkillMapper {
      constructor(schemaConverter?: SchemaConverter)

      // Returns AgentSkill, or null if module has no description.
      toSkill(descriptor: ModuleDescriptor, moduleId?: string): AgentSkill | null

      // "image.resize" -> "Image Resize"
      humanizeModuleId(moduleId: string): string
    }
    ```

=== "Rust"

    ```rust
    // src/adapters/skill_mapper.rs
    impl SkillMapper {
        pub fn new() -> Self

        // Returns an AgentSkill built from the module descriptor.
        pub fn to_skill(
            &self,
            module_id: &str,
            descriptor: &ModuleDescriptor,
            description: &str,
        ) -> AgentSkill
    }
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

=== "Python"

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

=== "TypeScript"

    ```typescript
    // src/adapters/schema.ts — $ref inlining delegated to apcore-toolkit deepResolveRefs
    class SchemaConverter {
      // Convert input_schema: inline $refs, strip $defs, ensure root type=object.
      convertInputSchema(descriptor: ModuleDescriptor): JsonSchema

      // Convert output_schema for Artifact metadata annotation.
      convertOutputSchema(descriptor: ModuleDescriptor): JsonSchema

      // Return "string", "object", or "unknown".
      detectRootType(schema: JsonSchema): "string" | "object" | "unknown"
    }
    ```

=== "Rust"

    ```rust
    // src/adapters/schema.rs — $ref inlining delegated to apcore-toolkit deep_resolve_refs
    impl SchemaConverter {
        pub fn new() -> Self

        // Convert input_schema: inline $refs, strip $defs, ensure root type=object.
        pub fn convert_input_schema(&self, schema: &Value) -> Value

        // Convert output_schema for Artifact metadata annotation.
        pub fn convert_output_schema(&self, schema: &Value) -> Value

        // Return "string", "object", or "unknown".
        pub fn detect_root_type(&self, schema: Option<&Value>) -> &'static str
    }
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

=== "Python"

    ```python
    class ErrorMapper:
        def to_jsonrpc_error(self, error: Exception) -> dict:
            """Returns: {"code": int, "message": str}"""

        def _handle_apcore_error(self, error: Exception, error_code: str) -> dict:
            """Handle apcore errors with a .code attribute (string matching)."""

        def _sanitize_message(self, message: str) -> str:
            """Strip paths, tracebacks. Truncate to 500 chars."""
    ```

=== "TypeScript"

    ```typescript
    // src/adapters/errors.ts
    class ErrorMapper {
      // Returns { code, message }.
      toJsonRpcError(error: unknown): { code: number; message: string }

      // Formats an error for an error-formatter context (strips paths/tracebacks).
      format(error: unknown, context?: Record<string, unknown>): Record<string, unknown>
    }
    ```

=== "Rust"

    ```rust
    // src/adapters/errors.rs
    impl ErrorMapper {
        // Returns a JsonRpcError { code, message }.
        pub fn to_jsonrpc_error(error: &ModuleError) -> JsonRpcError
    }

    // Error-formatter registered under the "a2a" key (idempotent).
    pub struct A2aErrorFormatter; // impl ErrorFormatter; .format(error, context) -> Value
    pub fn register_a2a_error_formatter()
    ```

**Error dispatch:** Errors are matched by string `.code` attribute (e.g., `"MODULE_NOT_FOUND"`),
not by exception class names.

**Error map:**

| apcore `.code` | JSON-RPC code | message | sanitized |
|---|---|---|---|
| `MODULE_NOT_FOUND` | -32601 | sanitized original message | Yes |
| `SCHEMA_VALIDATION_ERROR` | -32602 | sanitized original message | Yes |
| `ACL_DENIED` | -32001 | `"Task not found"` | **Yes** (masks real type) |
| `MODULE_TIMEOUT` | -32603 | `"Execution timeout"` | No |
| `EXECUTION_CANCELLED` | -32603 | `"Execution cancelled"` | No |
| `GENERAL_INVALID_INPUT` | -32602 | `"Invalid input: {sanitized description}"` | Yes |
| `CALL_DEPTH_EXCEEDED` / `CIRCULAR_CALL` / `CALL_FREQUENCY_EXCEEDED` | -32603 | `"Safety limit exceeded"` | No |
| `CIRCUIT_BREAKER_OPEN` / `TASK_LIMIT_EXCEEDED` | -32603 | `"Service temporarily unavailable"` | No |
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

=== "Python"

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

=== "TypeScript"

    ```typescript
    // src/adapters/parts.ts
    class PartConverter {
      // schemaConverter defaults to a new SchemaConverter() if not provided.
      constructor(schemaConverter?: SchemaConverter)

      // Convert A2A message Parts to apcore module input.
      // Empty/multiple parts and FileParts throw; DataPart → data object;
      // TextPart → string (root type 'string') or JSON.parse (root type 'object').
      partsToInput(parts: Part[], descriptor: ModuleDescriptor | null): Record<string, unknown> | string

      // Convert apcore module output to an A2A Artifact (object → DataPart, else TextPart).
      outputToParts(output: unknown, taskId?: string): Artifact
    }
    ```

=== "Rust"

    ```rust
    // src/adapters/parts.rs
    impl PartConverter {
        pub fn new(schema_converter: SchemaConverter) -> Self

        // Convert A2A message Parts to apcore module input.
        // Empty/multiple parts and FileParts return Err; DataPart → data object;
        // TextPart → string (root type "string") or parsed JSON (root type "object").
        pub fn parts_to_input(&self, parts: &[Part], input_schema: Option<&Value>) -> Result<Value, String>

        // Convert apcore module output to an A2A Artifact.
        pub fn output_to_parts(&self, output: &Value, task_id: &str) -> Artifact

        // Convert a raw result Value into a Vec<Part>.
        pub fn convert_result(&self, result: &Value) -> Vec<Part>
    }
    ```

> **A2A 1.0 Part shape.** `Part` is a protobuf `oneof`: a text part carries
> `content = text`, a data part carries `content = data`. On the wire it
> serializes flat — `{"text": "..."}` or `{"data": {...}}` — with no `type`
> discriminator. Reading inspects the active oneof case (Python:
> `part.WhichOneof("content")`; TypeScript: `part.content.$case`; Rust: the
> `Part` enum variant). A "FilePart" corresponds to the `raw` (bytes) or `url`
> oneof case and is rejected as unsupported.

## File Structure

=== "Python"

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

=== "TypeScript"

    ```
    src/adapters/
        agent-card.ts        # AgentCardBuilder
        skill-mapper.ts      # SkillMapper
        schema.ts            # SchemaConverter
        errors.ts            # ErrorMapper
        parts.ts             # PartConverter
    # exported from src/index.ts: AgentCardBuilder, SkillMapper, SchemaConverter,
    #                             ErrorMapper, PartConverter
    ```

=== "Rust"

    ```
    src/adapters/
        agent_card.rs        # AgentCardBuilder
        skill_mapper.rs      # SkillMapper
        schema.rs            # SchemaConverter
        errors.rs            # ErrorMapper, A2aErrorFormatter, register_a2a_error_formatter
        parts.rs             # PartConverter
    // re-exported from src/lib.rs: AgentCardBuilder, SkillMapper, SchemaConverter,
    //                             ErrorMapper, PartConverter, AdapterError
    ```

## Key Invariants

- No I/O (no network, no file system, no subprocess)
- All methods are pure or nearly-pure (deterministic given inputs)
- Deep-copy all input schemas before mutation
- `to_skill()` returns `None` for invalid modules (never raises)
- Error sanitization: never expose caller identity, file paths, or stack traces to callers

## Test Module

`tests/adapters/` — all unit tests, no fixtures beyond stubs
