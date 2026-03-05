# Test Plan: apcore-a2a

| Field       | Value                                                                 |
|-------------|-----------------------------------------------------------------------|
| Title       | apcore-a2a: Test Plan and Test Cases                                  |
| Document    | Test Plan (TP)                                                        |
| Document ID | TP-APCORE-A2A-001                                                     |
| Version     | 1.0                                                                   |
| Date        | 2026-03-03                                                            |
| Author      | aipartnerup Engineering Team                                          |
| Status      | Draft                                                                 |
| SRS Ref     | `docs/apcore-a2a/srs.md` v1.0 (SRS-APCORE-A2A-001)                  |
| Tech Design | `docs/apcore-a2a/tech-design.md` v1.0 (TDD-APCORE-A2A-001)          |
| Standard    | IEEE 829-2008, ISTQB Foundation Level                                 |

---

## Revision History

| Version | Date       | Author                       | Description   |
|---------|------------|------------------------------|---------------|
| 1.0     | 2026-03-03 | aipartnerup Engineering Team | Initial draft |

---

## Table of Contents

1. [Test Plan Overview](#1-test-plan-overview)
2. [Test Strategy](#2-test-strategy)
3. [Test Environment](#3-test-environment)
4. [Test Schedule](#4-test-schedule)
5. [Unit Test Cases — Adapters](#5-unit-test-cases--adapters)
6. [Unit Test Cases — Server](#6-unit-test-cases--server)
7. [Unit Test Cases — Client & Auth & Storage](#7-unit-test-cases--client--auth--storage)
8. [Unit Test Cases — CLI, Explorer, Ops](#8-unit-test-cases--cli-explorer-ops)
9. [Integration Tests](#9-integration-tests)
10. [End-to-End Tests](#10-end-to-end-tests)
11. [Performance Tests](#11-performance-tests)
12. [Security Tests](#12-security-tests)
13. [Test Data and Fixtures](#13-test-data-and-fixtures)
14. [Traceability Matrix](#14-traceability-matrix)
15. [Quality Gates](#15-quality-gates)
16. [Risk Assessment](#16-risk-assessment)

---

## 1. Test Plan Overview

### 1.1 Purpose

This Test Plan defines the complete testing strategy, test cases, and acceptance criteria for **apcore-a2a v1.0.0**. It covers all 19 PRD features formalized into 63 functional requirements and 30 non-functional requirements in the upstream SRS. Every test case is traceable to SRS requirements and the technical design.

### 1.2 Scope

**In scope:**
- All functional requirements (FR-SRV, FR-AGC, FR-SKL, FR-MSG, FR-TSK, FR-EXE, FR-ERR, FR-CLI, FR-CTX, FR-PSH, FR-AUT, FR-STR, FR-CMD, FR-EXP, FR-OPS)
- All non-functional requirements (NFR-PERF, NFR-SEC, NFR-REL, NFR-CMP, NFR-MNT, NFR-SCA)
- A2A Client module (A2AClient, AgentCardFetcher)
- All API endpoints: JSON-RPC methods, REST endpoints, SSE streams, webhooks

**Out of scope:**
- gRPC binding (not implemented in v1)
- Agent Card cryptographic signing (deferred)
- Persistent storage backends beyond InMemoryTaskStore
- Browser-based manual Explorer UI testing (covered by automated endpoint tests)

### 1.3 Quality Objectives

1. Line coverage ≥ 90% (matching apcore-mcp-python standard)
2. Branch coverage ≥ 85%
3. Zero P0 test cases failing at release
4. All SRS functional requirements have ≥ 1 test case
5. All NFR requirements have ≥ 1 measurable test

---

## 2. Test Strategy

### 2.1 Test Levels

| Level | Tool | Scope | Mock Boundary |
|-------|------|-------|---------------|
| **Unit** | pytest + pytest-asyncio | Individual components in isolation | Registry, Executor (stub) |
| **Integration** | pytest + pytest-asyncio + httpx | Cross-component flows | External A2A clients |
| **E2E** | pytest + httpx + real server | Full system via HTTP | None |
| **Performance** | pytest + time/asyncio | Latency and throughput | None |
| **Security** | pytest + httpx | Auth, sanitization | None |

### 2.2 Test Framework Configuration

```toml
# pyproject.toml (test section)
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
filterwarnings = ["error"]

[tool.coverage.run]
source = ["src/apcore_a2a"]
branch = true

[tool.coverage.report]
fail_under = 90
```

### 2.3 Stub Pattern

All unit tests use lightweight stub fixtures — NOT the real apcore-python SDK:

```python
# conftest.py pattern
from dataclasses import dataclass, field

@dataclass
class StubAnnotations:
    readonly: bool = False
    destructive: bool = False
    idempotent: bool = False
    requires_approval: bool = False
    open_world: bool = True

@dataclass
class StubExample:
    title: str
    inputs: dict = field(default_factory=dict)

@dataclass
class StubDescriptor:
    module_id: str
    description: str = ""
    tags: list[str] = field(default_factory=list)
    examples: list[StubExample] = field(default_factory=list)
    input_schema: dict = field(default_factory=dict)
    output_schema: dict = field(default_factory=dict)
    annotations: StubAnnotations = field(default_factory=StubAnnotations)

class StubRegistry:
    def __init__(self, modules: dict[str, StubDescriptor]):
        self._modules = modules
    def list(self, tags=None, prefix=None) -> list[str]:
        return list(self._modules.keys())
    def get_definition(self, module_id: str) -> StubDescriptor | None:
        return self._modules.get(module_id)

class StubExecutor:
    def __init__(self, results: dict[str, any] = None, errors: dict[str, Exception] = None):
        self._results = results or {}
        self._errors = errors or {}
    async def call_async(self, module_id: str, inputs: dict) -> any:
        if module_id in self._errors:
            raise self._errors[module_id]
        return self._results.get(module_id, {"result": "ok"})
    async def stream(self, module_id: str, inputs: dict):
        for chunk in self._results.get(module_id, [{"chunk": 1}]):
            yield chunk
    def validate(self, module_id: str, inputs: dict) -> None:
        pass
```

### 2.4 Automation

All 100% automated via pytest. No manual test cases. CI runs on every PR via GitHub Actions.

---

## 3. Test Environment

### 3.1 Software Requirements

| Component | Version |
|-----------|---------|
| Python | 3.11+ |
| pytest | ≥ 8.0 |
| pytest-asyncio | ≥ 0.23 |
| pytest-cov | ≥ 4.1 |
| httpx | ≥ 0.27 (async HTTP client for integration/E2E) |
| PyJWT | ≥ 2.8 |
| ruff | ≥ 0.3 |
| mypy | ≥ 1.9 |

### 3.2 Test Directory Structure

```
tests/
├── conftest.py                # Shared stubs and fixtures
├── adapters/
│   ├── test_agent_card_builder.py
│   ├── test_skill_mapper.py
│   ├── test_schema_converter.py
│   ├── test_part_converter.py
│   └── test_error_mapper.py
├── server/
│   ├── test_server_factory.py
│   ├── test_execution_router.py
│   ├── test_task_manager.py
│   └── test_streaming_handler.py
├── push/
│   └── test_push_notification_manager.py
├── client/
│   ├── test_a2a_client.py
│   └── test_agent_card_fetcher.py
├── auth/
│   ├── test_jwt_authenticator.py
│   └── test_auth_middleware.py
├── storage/
│   └── test_in_memory_task_store.py
├── cli/
│   └── test_cli.py
├── explorer/
│   └── test_explorer.py
├── ops/
│   └── test_operations.py
├── integration/
│   └── test_integration.py
├── e2e/
│   └── test_e2e.py
├── performance/
│   └── test_performance.py
└── security/
    └── test_security.py
```

### 3.3 CI/CD Integration

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -e ".[dev]"
      - run: ruff check src/ tests/
      - run: mypy src/
      - run: pytest --cov=apcore_a2a --cov-report=xml --cov-fail-under=90
```

---

## 4. Test Schedule

| Phase | Test Types | Duration | Entry Criteria |
|-------|-----------|----------|----------------|
| Phase 1 — Adapters | Unit (TC-AGC, TC-SKL, TC-SCH, TC-PRT, TC-ERR) | Week 1-2 | Components implemented |
| Phase 2 — Server Core | Unit (TC-SRV, TC-RTR, TC-TSK, TC-STR) | Week 3-4 | Adapter tests passing |
| Phase 3 — Push/Client/Auth | Unit (TC-PSH, TC-CLI, TC-AUT, TC-STO) | Week 5-6 | Server core tests passing |
| Phase 4 — Integration | TC-INT | Week 7 | All unit tests passing |
| Phase 5 — E2E/Perf/Security | TC-E2E, TC-PERF, TC-SEC | Week 8-9 | Integration tests passing |

---

## 5. Unit Test Cases — Adapters

### 5.1 TC-AGC: Agent Card Builder

---

#### TC-AGC-001: Generate Agent Card with all Registry metadata fields

| Field | Value |
|-------|-------|
| **ID** | TC-AGC-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-AGC-001 |

**Preconditions:** Registry with project config `{name: "my-agent", description: "Test agent", version: "1.2.3"}` and 2 modules: `math.add` (description: "Adds two numbers") and `math.multiply` (description: "Multiplies two numbers").

**Test Steps:**
1. Create `StubRegistry` with the above 2 modules.
2. Instantiate `AgentCardBuilder(registry, url="http://localhost:8000")`.
3. Call `builder.build()`.
4. Assert `card.name == "my-agent"`.
5. Assert `card.description == "Test agent"`.
6. Assert `card.version == "1.2.3"`.
7. Assert `card.url == "http://localhost:8000"`.
8. Assert `len(card.skills) == 2`.
9. Assert `"application/json"` in `card.defaultInputModes`.
10. Assert `"application/json"` in `card.defaultOutputModes`.

**Expected Result:** Agent Card built successfully with all fields matching Registry config.

---

#### TC-AGC-002: Fallback defaults when Registry has no project config

| Field | Value |
|-------|-------|
| **ID** | TC-AGC-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-AGC-001 |

**Preconditions:** Registry with no project config, 3 modules.

**Test Steps:**
1. Create `StubRegistry` with no project config and 3 modules.
2. Call `AgentCardBuilder(registry, url="http://localhost:9000").build()`.
3. Assert `card.name == "apcore-agent"`.
4. Assert `card.version == "0.0.0"`.
5. Assert `"3" in card.description` (auto-generated includes module count).

**Expected Result:** Fallbacks applied; description mentions "3 skills".

---

#### TC-AGC-003: Capabilities set to True when streaming module present

| Field | Value |
|-------|-------|
| **ID** | TC-AGC-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-AGC-002 |

**Preconditions:** Registry with one streaming-capable module (executor has `stream()` method).

**Test Steps:**
1. Create `StubExecutor` that has a working `stream()` coroutine.
2. Build Agent Card with `push_notifications=True`.
3. Assert `card.capabilities.streaming == True`.
4. Assert `card.capabilities.pushNotifications == True`.

**Expected Result:** Both capabilities are True.

---

#### TC-AGC-004: capabilities.streaming is False when no streaming modules

| Field | Value |
|-------|-------|
| **ID** | TC-AGC-004 |
| **Priority** | P0 |
| **SRS Trace** | FR-AGC-002 |

**Preconditions:** Registry with only non-streaming modules, `push_notifications=False`.

**Test Steps:**
1. Create `StubExecutor` without `stream()` method.
2. Build Agent Card.
3. Assert `card.capabilities.streaming == False`.
4. Assert `card.capabilities.pushNotifications == False`.

**Expected Result:** Both capabilities are False.

---

#### TC-AGC-005: GET /.well-known/agent.json returns 200 with valid JSON

| Field | Value |
|-------|-------|
| **ID** | TC-AGC-005 |
| **Priority** | P0 |
| **SRS Trace** | FR-AGC-003 |

**Preconditions:** A2A server running with 1 module `data.echo`.

**Test Steps:**
1. Start test ASGI client using `httpx.AsyncClient(app=asgi_app)`.
2. Send `GET /.well-known/agent.json`.
3. Assert status code 200.
4. Assert `Content-Type: application/json`.
5. Assert `Cache-Control` header contains `max-age=300`.
6. Parse body as JSON — assert `card["name"]` is a non-empty string.
7. Assert `card["skills"]` is a list with 1 item.

**Expected Result:** HTTP 200, valid Agent Card JSON with `Cache-Control: max-age=300`.

---

#### TC-AGC-006: /.well-known/agent.json requires NO authentication

| Field | Value |
|-------|-------|
| **ID** | TC-AGC-006 |
| **Priority** | P0 |
| **SRS Trace** | FR-AGC-003 |

**Preconditions:** Server configured with JWT auth (`require_auth=True`).

**Test Steps:**
1. Start server with `JWTAuthenticator(secret="test-secret")` and `require_auth=True`.
2. Send `GET /.well-known/agent.json` with NO `Authorization` header.
3. Assert status code 200.

**Expected Result:** 200 (public endpoint, auth-exempt).

---

#### TC-AGC-007: Extended card at /agent/authenticatedExtendedCard requires auth

| Field | Value |
|-------|-------|
| **ID** | TC-AGC-007 |
| **Priority** | P1 |
| **SRS Trace** | FR-AGC-004 |

**Preconditions:** Server with `JWTAuthenticator(secret="secret-key-123")`.

**Test Steps:**
1. Send `GET /agent/authenticatedExtendedCard` without auth header.
2. Assert status code 401.
3. Send again with `Authorization: Bearer {valid_jwt}`.
4. Assert status code 200.
5. Assert response body is a valid Agent Card JSON.

**Expected Result:** 401 without auth, 200 with valid JWT.

---

#### TC-AGC-008: Agent Card updated within 1s after module registration

| Field | Value |
|-------|-------|
| **ID** | TC-AGC-008 |
| **Priority** | P2 |
| **SRS Trace** | FR-AGC-005 |

**Preconditions:** Running server, registry initially has 1 module.

**Test Steps:**
1. Fetch Agent Card — assert 1 skill.
2. Call `registry.register(new_module)`.
3. `await asyncio.sleep(1.1)`.
4. Fetch Agent Card again.
5. Assert 2 skills in updated card.

**Expected Result:** Card updated within 1 second.

---

### 5.2 TC-SKL: Skill Mapper

---

#### TC-SKL-001: Module ID maps exactly to Skill.id

| Field | Value |
|-------|-------|
| **ID** | TC-SKL-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-SKL-001 |

**Preconditions:** Module with `module_id = "image.resize"`.

**Test Steps:**
1. Call `SkillMapper.to_skill(descriptor)`.
2. Assert `skill.id == "image.resize"`.

**Expected Result:** `Skill.id == "image.resize"` (no transformation).

---

#### TC-SKL-002: Module description maps verbatim to Skill.description

| Field | Value |
|-------|-------|
| **ID** | TC-SKL-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-SKL-001 |

**Preconditions:** Module with `description = "Resizes an image to specified dimensions"`.

**Test Steps:**
1. Map descriptor.
2. Assert `skill.description == "Resizes an image to specified dimensions"`.

**Expected Result:** Description preserved verbatim.

---

#### TC-SKL-003: Module with empty description excluded from skills

| Field | Value |
|-------|-------|
| **ID** | TC-SKL-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-SKL-001 |

**Preconditions:** Module with `description = ""` (empty string).

**Test Steps:**
1. Call `SkillMapper.to_skill(descriptor)`.
2. Assert result is `None`.
3. Assert warning was logged containing `module_id`.

**Expected Result:** Returns `None`, warning logged.

---

#### TC-SKL-004: Module with None description excluded from skills

| Field | Value |
|-------|-------|
| **ID** | TC-SKL-004 |
| **Priority** | P0 |
| **SRS Trace** | FR-SKL-001 |

**Preconditions:** Module with `description = None`.

**Test Steps:**
1. Call `SkillMapper.to_skill(descriptor)`.
2. Assert result is `None`.

**Expected Result:** Returns `None`.

---

#### TC-SKL-005: Module tags map to Skill.tags

| Field | Value |
|-------|-------|
| **ID** | TC-SKL-005 |
| **Priority** | P0 |
| **SRS Trace** | FR-SKL-001 |

**Preconditions:** Module with `tags = ["image", "resize", "utility"]`.

**Test Steps:**
1. Map descriptor.
2. Assert `skill.tags == ["image", "resize", "utility"]`.

**Expected Result:** Tags list identical.

---

#### TC-SKL-006: Examples mapped correctly, max 10 included

| Field | Value |
|-------|-------|
| **ID** | TC-SKL-006 |
| **Priority** | P0 |
| **SRS Trace** | FR-SKL-002 |

**Preconditions:** Module with 12 examples, first has `title = "Resize to 512x512"`, `inputs = {"width": 512, "height": 512}`.

**Test Steps:**
1. Map descriptor.
2. Assert `len(skill.examples) == 10`.
3. Assert `skill.examples[0].name == "Resize to 512x512"`.
4. Assert first example input contains JSON serialization of `{"width": 512, "height": 512}`.

**Expected Result:** Only first 10 examples included, name and input correct.

---

#### TC-SKL-007: Module with input_schema declares application/json inputMode

| Field | Value |
|-------|-------|
| **ID** | TC-SKL-007 |
| **Priority** | P0 |
| **SRS Trace** | FR-SKL-003 |

**Preconditions:** Module with `input_schema = {"type": "object", "properties": {"n": {"type": "integer"}}}`.

**Test Steps:**
1. Map descriptor.
2. Assert `"application/json"` in `skill.inputModes`.

**Expected Result:** `inputModes` contains `"application/json"`.

---

#### TC-SKL-008: Module with no input_schema declares text/plain inputMode

| Field | Value |
|-------|-------|
| **ID** | TC-SKL-008 |
| **Priority** | P0 |
| **SRS Trace** | FR-SKL-003 |

**Preconditions:** Module with `input_schema = None` or `{}`.

**Test Steps:**
1. Map descriptor.
2. Assert `"text/plain"` in `skill.inputModes`.
3. Assert `"application/json"` NOT in `skill.inputModes`.

**Expected Result:** Only `text/plain` in inputModes.

---

#### TC-SKL-009: Annotations included as Skill extensions

| Field | Value |
|-------|-------|
| **ID** | TC-SKL-009 |
| **Priority** | P0 |
| **SRS Trace** | FR-SKL-004 |

**Preconditions:** Module with `annotations = StubAnnotations(readonly=True, destructive=False)`.

**Test Steps:**
1. Map descriptor.
2. Assert `skill.extensions["apcore"]["annotations"]["readonly"] == True`.
3. Assert `skill.extensions["apcore"]["annotations"]["destructive"] == False`.

**Expected Result:** Annotations in extensions with correct boolean values.

---

#### TC-SKL-010: Modules with None annotations omit extensions key

| Field | Value |
|-------|-------|
| **ID** | TC-SKL-010 |
| **Priority** | P0 |
| **SRS Trace** | FR-SKL-004 |

**Preconditions:** Module with `annotations = None`.

**Test Steps:**
1. Map descriptor.
2. Assert `"apcore"` NOT in `skill.extensions` (or extensions is empty/None).

**Expected Result:** No apcore extension key.

---

### 5.3 TC-SCH: Schema Converter

---

#### TC-SCH-001: Empty schema converted to empty object schema

| Field | Value |
|-------|-------|
| **ID** | TC-SCH-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-EXE-003 |

**Preconditions:** Input schema `{}`.

**Test Steps:**
1. Call `converter.convert_input_schema(ModuleDescriptor(input_schema={}))`.
2. Assert result `== {"type": "object", "properties": {}}`.

**Expected Result:** `{"type": "object", "properties": {}}`.

---

#### TC-SCH-002: None schema converted to empty object schema

| Field | Value |
|-------|-------|
| **ID** | TC-SCH-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-EXE-003 |

**Test Steps:**
1. Call `converter.convert_input_schema(ModuleDescriptor(input_schema=None))`.
2. Assert result `== {"type": "object", "properties": {}}`.

**Expected Result:** `{"type": "object", "properties": {}}`.

---

#### TC-SCH-003: $ref resolved and $defs removed

| Field | Value |
|-------|-------|
| **ID** | TC-SCH-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-EXE-003 |

**Preconditions:**
```python
schema = {
    "type": "object",
    "$defs": {"Step": {"type": "object", "properties": {"name": {"type": "string"}}}},
    "properties": {"steps": {"type": "array", "items": {"$ref": "#/$defs/Step"}}}
}
```

**Test Steps:**
1. Call `converter.convert_input_schema(ModuleDescriptor(input_schema=schema))`.
2. Assert `"$defs"` NOT in result.
3. Assert `result["properties"]["steps"]["items"] == {"type": "object", "properties": {"name": {"type": "string"}}}`.

**Expected Result:** `$ref` inlined, `$defs` removed.

---

#### TC-SCH-004: Circular $ref raises ValueError

| Field | Value |
|-------|-------|
| **ID** | TC-SCH-004 |
| **Priority** | P0 |
| **SRS Trace** | FR-EXE-003 |

**Preconditions:**
```python
schema = {
    "$defs": {"A": {"$ref": "#/$defs/B"}, "B": {"$ref": "#/$defs/A"}},
    "properties": {"x": {"$ref": "#/$defs/A"}}
}
```

**Test Steps:**
1. Call `converter.convert_input_schema(ModuleDescriptor(input_schema=schema))`.
2. Assert `ValueError` is raised.
3. Assert error message contains "circular" or "recursion".

**Expected Result:** `ValueError` raised with descriptive message.

---

#### TC-SCH-005: Input schema deep-copied (original not mutated)

| Field | Value |
|-------|-------|
| **ID** | TC-SCH-005 |
| **Priority** | P0 |
| **SRS Trace** | FR-EXE-003 |

**Preconditions:** Schema with `$defs` and nested properties.

**Test Steps:**
1. Store original schema.
2. Call `converter.convert_input_schema(ModuleDescriptor(input_schema=schema))`.
3. Assert original schema still has `$defs` key.

**Expected Result:** Original schema unchanged.

---

### 5.4 TC-PRT: Part Converter

---

#### TC-PRT-001: Dict output converted to DataPart with application/json

| Field | Value |
|-------|-------|
| **ID** | TC-PRT-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-004 |

**Preconditions:** Module output `{"result": 42, "status": "ok"}`.

**Test Steps:**
1. Call `PartConverter.output_to_parts({"result": 42, "status": "ok"})`.
2. Assert artifact has one `DataPart`.
3. Assert `part.mediaType == "application/json"`.
4. Assert `part.data == {"result": 42, "status": "ok"}`.

**Expected Result:** DataPart with application/json media type.

---

#### TC-PRT-002: String output converted to TextPart

| Field | Value |
|-------|-------|
| **ID** | TC-PRT-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-004 |

**Preconditions:** Module output `"Hello, world!"`.

**Test Steps:**
1. Call `PartConverter.output_to_parts("Hello, world!")`.
2. Assert artifact has one `TextPart`.
3. Assert `part.text == "Hello, world!"`.

**Expected Result:** TextPart with correct text.

---

#### TC-PRT-003: DataPart with application/json parsed to dict input

| Field | Value |
|-------|-------|
| **ID** | TC-PRT-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-003 |

**Preconditions:** A2A message with `DataPart(mediaType="application/json", data={"width": 512, "height": 512})`.

**Test Steps:**
1. Call `PartConverter.parts_to_input(parts, input_schema)`.
2. Assert result `== {"width": 512, "height": 512}`.

**Expected Result:** Dict input extracted from DataPart.

---

#### TC-PRT-004: TextPart with valid JSON auto-parsed when module expects object

| Field | Value |
|-------|-------|
| **ID** | TC-PRT-004 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-003 |

**Preconditions:** TextPart with `text = '{"width": 256, "height": 256}'`, input_schema root type `"object"`.

**Test Steps:**
1. Call `PartConverter.parts_to_input(parts, input_schema={"type": "object"})`.
2. Assert result `== {"width": 256, "height": 256}`.

**Expected Result:** JSON auto-parsed from TextPart.

---

#### TC-PRT-005: TextPart with invalid JSON raises JSON-RPC -32602

| Field | Value |
|-------|-------|
| **ID** | TC-PRT-005 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-003 |

**Preconditions:** TextPart with `text = "not json {"`, module expects object input.

**Test Steps:**
1. Call `PartConverter.parts_to_input(parts, input_schema={"type": "object"})`.
2. Assert `A2AError` raised with code `-32602`.
3. Assert message contains `"Invalid JSON in TextPart"`.

**Expected Result:** Error -32602 with "Invalid JSON in TextPart".

---

#### TC-PRT-006: Empty parts list raises JSON-RPC -32602

| Field | Value |
|-------|-------|
| **ID** | TC-PRT-006 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-003 |

**Test Steps:**
1. Call `PartConverter.parts_to_input([], input_schema={})`.
2. Assert `A2AError` raised with code `-32602`.
3. Assert message contains `"Message must contain at least one Part"`.

**Expected Result:** Error -32602 with required message text.

---

### 5.5 TC-ERR: Error Mapper

---

#### TC-ERR-001: ModuleNotFoundError maps to JSON-RPC -32601

| Field | Value |
|-------|-------|
| **ID** | TC-ERR-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-ERR-001 |

**Test Steps:**
1. Call `ErrorMapper.to_jsonrpc_error(ModuleNotFoundError("unknown.module"))`.
2. Assert `result.code == -32601`.

**Expected Result:** Code -32601.

---

#### TC-ERR-002: SchemaValidationError maps to JSON-RPC -32602

| Field | Value |
|-------|-------|
| **ID** | TC-ERR-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-ERR-001 |

**Test Steps:**
1. Call `ErrorMapper.to_jsonrpc_error(SchemaValidationError({"width": "must be integer"}))`.
2. Assert `result.code == -32602`.
3. Assert `"width"` in `result.message`.

**Expected Result:** Code -32602 with field name in message.

---

#### TC-ERR-003: ACLDeniedError maps to -32001 with sanitized message

| Field | Value |
|-------|-------|
| **ID** | TC-ERR-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-ERR-001, NFR-SEC-003 |

**Test Steps:**
1. Call `ErrorMapper.to_jsonrpc_error(ACLDeniedError(caller="admin", module="secret.module"))`.
2. Assert `result.code == -32001`.
3. Assert `"admin"` NOT in `result.message` (caller info sanitized).
4. Assert `"secret.module"` NOT in `result.message` (module name sanitized).

**Expected Result:** Code -32001, no internal details in message.

---

#### TC-ERR-004: ModuleExecuteError maps to -32603 with sanitized message

| Field | Value |
|-------|-------|
| **ID** | TC-ERR-004 |
| **Priority** | P0 |
| **SRS Trace** | FR-ERR-001 |

**Test Steps:**
1. Call `ErrorMapper.to_jsonrpc_error(ModuleExecuteError("Database connection failed at 192.168.1.1"))`.
2. Assert `result.code == -32603`.
3. Assert `"192.168.1.1"` NOT in `result.message`.

**Expected Result:** Code -32603, internal IP not leaked.

---

#### TC-ERR-005: Unknown exception type maps to -32603

| Field | Value |
|-------|-------|
| **ID** | TC-ERR-005 |
| **Priority** | P0 |
| **SRS Trace** | FR-ERR-001 |

**Test Steps:**
1. Call `ErrorMapper.to_jsonrpc_error(RuntimeError("unexpected crash"))`.
2. Assert `result.code == -32603`.
3. Assert `result.message` is a generic error message (not the raw exception message).

**Expected Result:** Code -32603, generic message.

---

## 6. Unit Test Cases — Server

### 6.1 TC-SRV: Server Factory

---

#### TC-SRV-001: Server factory creates ASGI app with all required routes

| Field | Value |
|-------|-------|
| **ID** | TC-SRV-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-SRV-001, FR-SRV-004 |

**Test Steps:**
1. Create `A2AServerFactory().create(registry=stub_registry, executor=stub_executor)`.
2. Assert ASGI app has route `GET /.well-known/agent.json`.
3. Assert ASGI app has route `POST /` for JSON-RPC.
4. Assert ASGI app has route `GET /agent/authenticatedExtendedCard` (even if 404 without auth config).

**Expected Result:** All required routes present.

---

#### TC-SRV-002: serve() raises ValueError when registry has zero modules

| Field | Value |
|-------|-------|
| **ID** | TC-SRV-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-SRV-001 |

**Test Steps:**
1. Create `StubRegistry({})` (no modules).
2. Call `serve(registry)`.
3. Assert `ValueError` raised with message mentioning "zero modules" or "no modules".

**Expected Result:** `ValueError` raised.

---

#### TC-SRV-003: async_serve() returns ASGI app without starting server

| Field | Value |
|-------|-------|
| **ID** | TC-SRV-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-SRV-003 |

**Test Steps:**
1. Call `app = await async_serve(registry)`.
2. Assert `app` is an ASGI-callable (has `__call__` with `scope, receive, send` params).
3. Assert no server socket is bound on port 8080.

**Expected Result:** ASGI app returned, no port bound.

---

#### TC-SRV-004: Malformed JSON returns -32700

| Field | Value |
|-------|-------|
| **ID** | TC-SRV-004 |
| **Priority** | P0 |
| **SRS Trace** | FR-SRV-004 |

**Test Steps:**
1. POST `/` with body `"not valid json {"`.
2. Assert HTTP status 200 (JSON-RPC errors are returned as HTTP 200).
3. Parse response — assert `error.code == -32700`.

**Expected Result:** JSON-RPC error -32700.

---

#### TC-SRV-005: Missing jsonrpc field returns -32600

| Field | Value |
|-------|-------|
| **ID** | TC-SRV-005 |
| **Priority** | P0 |
| **SRS Trace** | FR-SRV-004 |

**Test Steps:**
1. POST `/` with `{"method": "message/send", "id": 1, "params": {}}` (no `jsonrpc` field).
2. Assert `error.code == -32600`.

**Expected Result:** JSON-RPC error -32600.

---

#### TC-SRV-006: Unknown method returns -32601

| Field | Value |
|-------|-------|
| **ID** | TC-SRV-006 |
| **Priority** | P0 |
| **SRS Trace** | FR-SRV-004 |

**Test Steps:**
1. POST `/` with valid JSON-RPC but `method = "nonexistent/method"`.
2. Assert `error.code == -32601`.

**Expected Result:** JSON-RPC error -32601.

---

#### TC-SRV-007: Request body exceeding 10MB returns HTTP 413

| Field | Value |
|-------|-------|
| **ID** | TC-SRV-007 |
| **Priority** | P0 |
| **SRS Trace** | FR-SRV-004 |

**Test Steps:**
1. POST `/` with body of 11MB (padding in params).
2. Assert HTTP status code 413.

**Expected Result:** HTTP 413 (Payload Too Large).

---

### 6.2 TC-RTR: Execution Router

---

#### TC-RTR-001: Sync message routed to executor.call_async()

| Field | Value |
|-------|-------|
| **ID** | TC-RTR-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-EXE-001 |

**Preconditions:** `StubExecutor` with `results = {"math.add": {"sum": 7}}`.

**Test Steps:**
1. Create `ExecutionRouter(executor=stub_executor)`.
2. Call `result = await router.handle_message_send(skill_id="math.add", inputs={"a": 3, "b": 4}, context=Context())`.
3. Assert `result == {"sum": 7}`.
4. Assert `stub_executor.call_async` was called with `("math.add", {"a": 3, "b": 4})`.

**Expected Result:** Result `{"sum": 7}` returned.

---

#### TC-RTR-002: Streaming skill routed to executor.stream()

| Field | Value |
|-------|-------|
| **ID** | TC-RTR-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-EXE-001 |

**Preconditions:** `StubExecutor` with streaming output `[{"chunk": 1}, {"chunk": 2}, {"final": True}]`.

**Test Steps:**
1. Call `router.handle_message_stream(skill_id="data.stream", inputs={}, context=Context())`.
2. Collect all chunks from async generator.
3. Assert chunks == `[{"chunk": 1}, {"chunk": 2}, {"final": True}]`.

**Expected Result:** All 3 chunks yielded in order.

---

#### TC-RTR-003: Executor error propagated as correct exception type

| Field | Value |
|-------|-------|
| **ID** | TC-RTR-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-EXE-001, FR-ERR-001 |

**Preconditions:** `StubExecutor` raises `ModuleExecuteError("boom")` for `"fail.module"`.

**Test Steps:**
1. Call `await router.handle_message_send(skill_id="fail.module", inputs={}, context=Context())`.
2. Assert `ModuleExecuteError` (or wrapped `A2AError`) is raised.

**Expected Result:** Exception propagated correctly.

---

#### TC-RTR-004: ACLDeniedError sanitized before propagation

| Field | Value |
|-------|-------|
| **ID** | TC-RTR-004 |
| **Priority** | P0 |
| **SRS Trace** | FR-ERR-001, NFR-SEC-003 |

**Preconditions:** `StubExecutor` raises `ACLDeniedError(caller="user:bob", module="admin.delete")`.

**Test Steps:**
1. Call `await router.handle_message_send(skill_id="admin.delete", inputs={}, context=Context(identity=...))`.
2. Catch `A2AError` or equivalent.
3. Assert `"bob"` NOT in error message.
4. Assert `"admin.delete"` NOT in error message.

**Expected Result:** Sanitized error, no caller/module info leaked.

---

#### TC-RTR-005: Identity injected into apcore Context from AuthMiddleware

| Field | Value |
|-------|-------|
| **ID** | TC-RTR-005 |
| **Priority** | P1 |
| **SRS Trace** | FR-AUT-004 |

**Preconditions:** Auth middleware sets `Identity(id="user-123", type="user", roles=["viewer"])`.

**Test Steps:**
1. Route message with pre-set auth identity.
2. Capture `context` passed to `stub_executor.call_async`.
3. Assert `context.identity.id == "user-123"`.
4. Assert `context.identity.roles == ["viewer"]`.

**Expected Result:** Identity present in Context.

---

### 6.3 TC-TSK: Task Manager

---

#### TC-TSK-001: Task created with submitted state and UUID taskId

| Field | Value |
|-------|-------|
| **ID** | TC-TSK-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-TSK-001 |

**Test Steps:**
1. Call `task = await task_manager.create_task(context_id="ctx-abc", skill_id="math.add")`.
2. Assert `task.status.state == "submitted"`.
3. Assert `task.id` matches UUID v4 pattern `r'^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$'`.
4. Assert `task.context_id == "ctx-abc"`.

**Expected Result:** Task with submitted state and valid UUID.

---

#### TC-TSK-002: Valid state transitions succeed

| Field | Value |
|-------|-------|
| **ID** | TC-TSK-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-TSK-002 |

**Preconditions:** Task in `submitted` state.

**Test Steps:**
1. Transition submitted → working: `await task_manager.transition(task_id, "working")`. Assert succeeds.
2. Transition working → completed: assert succeeds.
3. Assert `task.status.state == "completed"`.

**Expected Result:** Both transitions succeed.

---

#### TC-TSK-003: Invalid state transition raises StateTransitionError

| Field | Value |
|-------|-------|
| **ID** | TC-TSK-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-TSK-003 |

**Preconditions:** Task in `completed` state.

**Test Steps:**
1. Attempt `await task_manager.transition(task_id, "working")`.
2. Assert `StateTransitionError` (or equivalent) is raised.

**Expected Result:** Exception raised on invalid transition.

---

#### TC-TSK-004: Completed task cannot transition to any state

| Field | Value |
|-------|-------|
| **ID** | TC-TSK-004 |
| **Priority** | P0 |
| **SRS Trace** | FR-TSK-003 |

**Preconditions:** Task in `completed` state.

**Test Steps:**
1. For each state in `["submitted", "working", "failed", "canceled", "input_required"]`:
   - Attempt transition.
   - Assert exception raised.

**Expected Result:** All 5 transitions raise exceptions.

---

#### TC-TSK-005: tasks/get returns correct task with history

| Field | Value |
|-------|-------|
| **ID** | TC-TSK-005 |
| **Priority** | P0 |
| **SRS Trace** | FR-TSK-005 |

**Preconditions:** Task `task-456` exists in completed state with 2 messages in history.

**Test Steps:**
1. POST JSON-RPC `tasks/get` with `params = {"id": "task-456", "historyLength": 5}`.
2. Assert response `result.id == "task-456"`.
3. Assert `result.status.state == "completed"`.
4. Assert `len(result.history) == 2`.

**Expected Result:** Task returned with correct state and history.

---

#### TC-TSK-006: tasks/get for unknown task returns -32001

| Field | Value |
|-------|-------|
| **ID** | TC-TSK-006 |
| **Priority** | P0 |
| **SRS Trace** | FR-TSK-005 |

**Test Steps:**
1. POST JSON-RPC `tasks/get` with `params = {"id": "nonexistent-uuid"}`.
2. Assert `error.code == -32001`.

**Expected Result:** Error -32001.

---

#### TC-TSK-007: tasks/cancel transitions working task to canceled

| Field | Value |
|-------|-------|
| **ID** | TC-TSK-007 |
| **Priority** | P0 |
| **SRS Trace** | FR-TSK-006 |

**Preconditions:** Task in `working` state.

**Test Steps:**
1. POST JSON-RPC `tasks/cancel` with `params = {"id": task_id}`.
2. Assert response task `status.state == "canceled"`.

**Expected Result:** Task canceled.

---

#### TC-TSK-008: tasks/cancel on completed task returns -32002

| Field | Value |
|-------|-------|
| **ID** | TC-TSK-008 |
| **Priority** | P0 |
| **SRS Trace** | FR-TSK-006 |

**Preconditions:** Task in `completed` state.

**Test Steps:**
1. POST `tasks/cancel`.
2. Assert `error.code == -32002`.

**Expected Result:** Error -32002 (TaskNotCancelable).

---

#### TC-TSK-009: Concurrent task creation generates unique IDs

| Field | Value |
|-------|-------|
| **ID** | TC-TSK-009 |
| **Priority** | P0 |
| **SRS Trace** | FR-TSK-001, NFR-SCA-001 |

**Test Steps:**
1. Create 100 tasks concurrently using `asyncio.gather()`.
2. Assert all 100 `task.id` values are unique.

**Expected Result:** 100 unique task IDs.

---

### 6.4 TC-STR: Streaming Handler

---

#### TC-STR-001: message/stream returns text/event-stream content type

| Field | Value |
|-------|-------|
| **ID** | TC-STR-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-002 |

**Test Steps:**
1. POST JSON-RPC `message/stream` for skill `"data.stream"`.
2. Assert response `Content-Type == "text/event-stream"`.

**Expected Result:** SSE content type header.

---

#### TC-STR-002: First SSE event is TaskStatusUpdateEvent with submitted state

| Field | Value |
|-------|-------|
| **ID** | TC-STR-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-002 |

**Test Steps:**
1. Start `message/stream` for a module.
2. Read first SSE event.
3. Parse event data as JSON-RPC response.
4. Assert `result.kind == "status-update"`.
5. Assert `result.status.state == "submitted"`.

**Expected Result:** First event has submitted state.

---

#### TC-STR-003: TaskArtifactUpdateEvent emitted for each stream chunk

| Field | Value |
|-------|-------|
| **ID** | TC-STR-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-002 |

**Preconditions:** Module produces 3 stream chunks: `{"part": 1}`, `{"part": 2}`, `{"part": 3}`.

**Test Steps:**
1. Start `message/stream`.
2. Collect all artifact-update events.
3. Assert 3 artifact-update events received.
4. Assert second and third events have `append: true`.
5. Assert last event has `lastChunk: true`.

**Expected Result:** 3 artifact events, correct append/lastChunk flags.

---

#### TC-STR-004: Final SSE event is TaskStatusUpdateEvent with completed state

| Field | Value |
|-------|-------|
| **ID** | TC-STR-004 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-002 |

**Test Steps:**
1. Collect all SSE events.
2. Assert last event `result.kind == "status-update"`.
3. Assert `result.status.state == "completed"`.
4. Assert `result.final == True`.

**Expected Result:** Last event is completed status update with `final: true`.

---

#### TC-STR-005: tasks/resubscribe reconnects to existing task stream

| Field | Value |
|-------|-------|
| **ID** | TC-STR-005 |
| **Priority** | P0 |
| **SRS Trace** | FR-TSK-007 |

**Preconditions:** Task `task-789` in `working` state.

**Test Steps:**
1. POST JSON-RPC `tasks/resubscribe` with `params = {"id": "task-789"}`.
2. Assert response `Content-Type == "text/event-stream"`.
3. Assert events received continue from current task state.

**Expected Result:** SSE stream re-established.

---

#### TC-STR-006: First SSE event emitted within 50ms

| Field | Value |
|-------|-------|
| **ID** | TC-STR-006 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-002, NFR-PERF-003 |

**Test Steps:**
1. Record `t0 = time.monotonic()` before sending `message/stream`.
2. Read first SSE event.
3. Record `t1 = time.monotonic()`.
4. Assert `(t1 - t0) * 1000 < 50`.

**Expected Result:** First event within 50ms.

---

## 7. Unit Test Cases — Client & Auth & Storage

### 7.1 TC-CLI: A2A Client

---

#### TC-CLI-001: Client discovers agent card from well-known URL

| Field | Value |
|-------|-------|
| **ID** | TC-CLI-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-CLI-001 |

**Preconditions:** Mock HTTP server at `http://test-agent.local` serving valid Agent Card JSON.

**Test Steps:**
1. Create `A2AClient(base_url="http://test-agent.local")`.
2. Call `card = await client.discover()`.
3. Assert `card.name` is non-empty string.
4. Assert `len(card.skills) >= 0`.

**Expected Result:** Agent Card returned.

---

#### TC-CLI-002: Client caches discovered Agent Card

| Field | Value |
|-------|-------|
| **ID** | TC-CLI-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-CLI-001 |

**Test Steps:**
1. Call `client.discover()` twice.
2. Assert mock HTTP server received only 1 GET request (cached on second call).

**Expected Result:** Second call served from cache.

---

#### TC-CLI-003: send_message returns Task on success

| Field | Value |
|-------|-------|
| **ID** | TC-CLI-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-CLI-002 |

**Preconditions:** Mock A2A server returns Task with `status.state == "completed"`.

**Test Steps:**
1. Call `task = await client.send_message(skill_id="echo.text", inputs={"text": "hello"})`.
2. Assert `task.status.state == "completed"`.
3. Assert `task.id` is a valid UUID string.

**Expected Result:** Completed Task returned.

---

#### TC-CLI-004: stream_message yields SSE events as async generator

| Field | Value |
|-------|-------|
| **ID** | TC-CLI-004 |
| **Priority** | P0 |
| **SRS Trace** | FR-CLI-003 |

**Preconditions:** Mock server returns 3 SSE events: submitted, artifact-update, completed.

**Test Steps:**
1. Call `events = client.stream_message(skill_id="data.process", inputs={})`.
2. Collect all events with `async for event in events`.
3. Assert 3 events received.
4. Assert events[0].kind == "status-update" and events[0].status.state == "submitted".
5. Assert events[2].status.state == "completed".

**Expected Result:** 3 events received in correct order.

---

#### TC-CLI-005: get_task retrieves task by ID

| Field | Value |
|-------|-------|
| **ID** | TC-CLI-005 |
| **Priority** | P0 |
| **SRS Trace** | FR-CLI-004 |

**Test Steps:**
1. Call `task = await client.get_task("task-uuid-001")`.
2. Assert `task.id == "task-uuid-001"`.

**Expected Result:** Task with matching ID.

---

#### TC-CLI-006: Connection refused raises ClientConnectionError

| Field | Value |
|-------|-------|
| **ID** | TC-CLI-006 |
| **Priority** | P0 |
| **SRS Trace** | FR-CLI-002 |

**Test Steps:**
1. Create `A2AClient(base_url="http://localhost:1")` (no server at port 1).
2. Call `await client.send_message(skill_id="any", inputs={})`.
3. Assert `A2AClientError` or `httpx.ConnectError` raised.

**Expected Result:** Connection error raised (no hang).

---

### 7.2 TC-AUT: Authentication

---

#### TC-AUT-001: Valid JWT accepted, identity populated

| Field | Value |
|-------|-------|
| **ID** | TC-AUT-001 |
| **Priority** | P1 |
| **SRS Trace** | FR-AUT-001 |

**Preconditions:** JWT signed with `secret = "test-secret-key-abc123"`, payload `{"sub": "user-42", "type": "user", "roles": ["admin"]}`.

**Test Steps:**
1. Generate JWT with above payload.
2. Call `auth.authenticate({"authorization": f"Bearer {token}"})`.
3. Assert `identity.id == "user-42"`.
4. Assert `identity.type == "user"`.
5. Assert `identity.roles == ["admin"]`.

**Expected Result:** Identity with sub, type, and roles.

---

#### TC-AUT-002: Expired JWT returns None identity

| Field | Value |
|-------|-------|
| **ID** | TC-AUT-002 |
| **Priority** | P1 |
| **SRS Trace** | FR-AUT-001 |

**Preconditions:** JWT with `exp = (now - 3600)` (expired 1 hour ago).

**Test Steps:**
1. Call `auth.authenticate({"authorization": f"Bearer {expired_token}"})`.
2. Assert result is `None`.

**Expected Result:** `None` returned (no exception).

---

#### TC-AUT-003: Invalid JWT signature returns None

| Field | Value |
|-------|-------|
| **ID** | TC-AUT-003 |
| **Priority** | P1 |
| **SRS Trace** | FR-AUT-001 |

**Preconditions:** JWT signed with `secret = "wrong-secret"`.

**Test Steps:**
1. Configure `JWTAuthenticator(secret="correct-secret")`.
2. Call `auth.authenticate({"authorization": f"Bearer {bad_token}"})`.
3. Assert result is `None`.

**Expected Result:** `None` returned.

---

#### TC-AUT-004: Missing Authorization header returns None

| Field | Value |
|-------|-------|
| **ID** | TC-AUT-004 |
| **Priority** | P1 |
| **SRS Trace** | FR-AUT-001 |

**Test Steps:**
1. Call `auth.authenticate({})` (empty headers).
2. Assert result is `None`.

**Expected Result:** `None` returned.

---

#### TC-AUT-005: AuthMiddleware with require_auth=True rejects unauthenticated requests

| Field | Value |
|-------|-------|
| **ID** | TC-AUT-005 |
| **Priority** | P1 |
| **SRS Trace** | FR-AUT-002 |

**Test Steps:**
1. Configure server with `JWTAuthenticator` and `require_auth=True`.
2. POST `/` JSON-RPC request without Authorization header.
3. Assert HTTP 401 response.

**Expected Result:** HTTP 401.

---

#### TC-AUT-006: Well-known endpoint exempt from auth even with require_auth=True

| Field | Value |
|-------|-------|
| **ID** | TC-AUT-006 |
| **Priority** | P1 |
| **SRS Trace** | FR-AUT-002, FR-AGC-003 |

**Test Steps:**
1. Configure server with `require_auth=True`.
2. GET `/.well-known/agent.json` without Authorization.
3. Assert HTTP 200.

**Expected Result:** HTTP 200 (exempt from auth).

---

### 7.3 TC-STO: Storage

---

#### TC-STO-001: InMemoryTaskStore CRUD operations

| Field | Value |
|-------|-------|
| **ID** | TC-STO-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-STR-001 |

**Test Steps:**
1. `store = InMemoryTaskStore()`.
2. Create task: `await store.save(task)` — assert no error.
3. Get task: `result = await store.get(task.id)` — assert `result.id == task.id`.
4. Update state: modify task.status, `await store.save(task)`.
5. Get again — assert updated state.
6. List: `tasks = await store.list(context_id=None)` — assert `len(tasks) == 1`.

**Expected Result:** All CRUD operations succeed.

---

#### TC-STO-002: Get non-existent task returns None

| Field | Value |
|-------|-------|
| **ID** | TC-STO-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-STR-001 |

**Test Steps:**
1. `result = await store.get("does-not-exist")`.
2. Assert `result is None`.

**Expected Result:** `None` returned.

---

#### TC-STO-003: Concurrent task updates are thread-safe

| Field | Value |
|-------|-------|
| **ID** | TC-STO-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-STR-002, NFR-REL-003 |

**Test Steps:**
1. Create task in store.
2. Launch 50 concurrent coroutines each updating `task.status.state` to different values.
3. Assert no exception raised.
4. Assert final `store.get(task.id)` returns a valid task (not corrupted).

**Expected Result:** No race condition, no corruption.

---

#### TC-STO-004: TaskStore protocol compliance check

| Field | Value |
|-------|-------|
| **ID** | TC-STO-004 |
| **Priority** | P0 |
| **SRS Trace** | FR-STR-001 |

**Test Steps:**
1. Assert `InMemoryTaskStore` implements all methods of `TaskStore` protocol: `save()`, `get()`, `list()`, `delete()`.
2. Assert each method signature matches protocol definition (using `inspect.signature`).

**Expected Result:** Protocol compliance confirmed.

---

### 7.4 TC-PSH: Push Notification Manager

---

#### TC-PSH-001: pushNotificationConfig/set stores webhook config

| Field | Value |
|-------|-------|
| **ID** | TC-PSH-001 |
| **Priority** | P1 |
| **SRS Trace** | FR-PSH-001 |

**Test Steps:**
1. POST JSON-RPC `tasks/pushNotificationConfig/set` with:
   ```json
   {"params": {"id": "task-001", "pushNotificationConfig": {"url": "https://webhook.example.com/notify", "token": "abc-token-123"}}}
   ```
2. Assert response is the stored `PushNotificationConfig`.
3. Assert `config.url == "https://webhook.example.com/notify"`.
4. Assert `config.token == "abc-token-123"`.

**Expected Result:** Config stored and returned.

---

#### TC-PSH-002: Push notification delivered via HTTP POST to webhook URL

| Field | Value |
|-------|-------|
| **ID** | TC-PSH-002 |
| **Priority** | P1 |
| **SRS Trace** | FR-PSH-002 |

**Preconditions:** Mock webhook server at `http://localhost:19999/hook`. Task with push config registered.

**Test Steps:**
1. Register push config for task with `url = "http://localhost:19999/hook"`.
2. Transition task to `completed`.
3. Assert mock webhook server received 1 POST request.
4. Assert request body contains `TaskStatusUpdateEvent` with `state == "completed"`.

**Expected Result:** Webhook POST received with correct event.

---

#### TC-PSH-003: Push notification includes X-A2A-Notification-Token header

| Field | Value |
|-------|-------|
| **ID** | TC-PSH-003 |
| **Priority** | P1 |
| **SRS Trace** | FR-PSH-002 |

**Preconditions:** Push config with `token = "verify-me-456"`.

**Test Steps:**
1. Trigger task state change.
2. Assert webhook request contains header `X-A2A-Notification-Token: verify-me-456`.

**Expected Result:** Token header present.

---

#### TC-PSH-004: Failed webhook delivery retried with exponential backoff

| Field | Value |
|-------|-------|
| **ID** | TC-PSH-004 |
| **Priority** | P1 |
| **SRS Trace** | FR-PSH-003 |

**Preconditions:** Mock webhook returns HTTP 500 for first 2 calls, HTTP 200 on 3rd.

**Test Steps:**
1. Trigger push notification.
2. Assert mock webhook received 3 POST requests total.
3. Assert time between attempts grows (backoff).

**Expected Result:** 3 attempts with increasing delay.

---

#### TC-PSH-005: push_notifications=False returns -32003 when client requests config

| Field | Value |
|-------|-------|
| **ID** | TC-PSH-005 |
| **Priority** | P1 |
| **SRS Trace** | FR-PSH-001 |

**Preconditions:** Server started with `push_notifications=False`.

**Test Steps:**
1. POST `tasks/pushNotificationConfig/set`.
2. Assert `error.code == -32003`.

**Expected Result:** Error -32003 (PushNotificationNotSupported).

---

## 8. Unit Test Cases — CLI, Explorer, Ops

### 8.1 TC-CMD: CLI

---

#### TC-CMD-001: CLI starts server with --extensions-dir argument

| Field | Value |
|-------|-------|
| **ID** | TC-CMD-001 |
| **Priority** | P1 |
| **SRS Trace** | FR-CMD-001 |

**Test Steps:**
1. Use `subprocess` or `click.testing.CliRunner`.
2. Run `apcore-a2a --extensions-dir /tmp/test-extensions --port 18001 --dry-run`.
3. Assert exit code 0.
4. Assert output contains server configuration summary.

**Expected Result:** CLI starts without error.

---

#### TC-CMD-002: CLI rejects non-existent extensions-dir

| Field | Value |
|-------|-------|
| **ID** | TC-CMD-002 |
| **Priority** | P1 |
| **SRS Trace** | FR-CMD-001 |

**Test Steps:**
1. Run `apcore-a2a --extensions-dir /nonexistent/path`.
2. Assert exit code 1.
3. Assert stderr contains path error message.

**Expected Result:** Exit code 1, error in stderr.

---

#### TC-CMD-003: CLI rejects invalid port number

| Field | Value |
|-------|-------|
| **ID** | TC-CMD-003 |
| **Priority** | P1 |
| **SRS Trace** | FR-CMD-001 |

**Test Steps:**
1. Run `apcore-a2a --extensions-dir /valid/path --port 99999`.
2. Assert exit code 1.
3. Assert error mentions invalid port.

**Expected Result:** Exit code 1, port validation error.

---

### 8.2 TC-EXP: Explorer

---

#### TC-EXP-001: Explorer HTML page returns 200

| Field | Value |
|-------|-------|
| **ID** | TC-EXP-001 |
| **Priority** | P2 |
| **SRS Trace** | FR-EXP-001 |

**Preconditions:** Server with `explorer=True`.

**Test Steps:**
1. GET `/explorer/`.
2. Assert HTTP 200.
3. Assert `Content-Type: text/html`.
4. Assert body contains `<html`.

**Expected Result:** HTML page served.

---

#### TC-EXP-002: Explorer skills list returns all skills

| Field | Value |
|-------|-------|
| **ID** | TC-EXP-002 |
| **Priority** | P2 |
| **SRS Trace** | FR-EXP-001 |

**Preconditions:** Server with 3 modules and `explorer=True`.

**Test Steps:**
1. GET `/explorer/skills`.
2. Assert HTTP 200.
3. Parse JSON — assert list of 3 skills.
4. Each skill has `name`, `description`, `tags` fields.

**Expected Result:** JSON list of 3 skills.

---

#### TC-EXP-003: Explorer is disabled when explorer=False

| Field | Value |
|-------|-------|
| **ID** | TC-EXP-003 |
| **Priority** | P2 |
| **SRS Trace** | FR-EXP-001 |

**Preconditions:** Server with `explorer=False` (default).

**Test Steps:**
1. GET `/explorer/`.
2. Assert HTTP 404.

**Expected Result:** HTTP 404.

---

### 8.3 TC-OPS: Operations

---

#### TC-OPS-001: Health endpoint returns status ok

| Field | Value |
|-------|-------|
| **ID** | TC-OPS-001 |
| **Priority** | P2 |
| **SRS Trace** | FR-OPS-001 |

**Test Steps:**
1. GET `/health`.
2. Assert HTTP 200.
3. Parse JSON — assert `body["status"] == "ok"`.
4. Assert `body["module_count"]` is an integer >= 1.

**Expected Result:** `{"status": "ok", "module_count": N}`.

---

#### TC-OPS-002: Metrics endpoint returns Prometheus format

| Field | Value |
|-------|-------|
| **ID** | TC-OPS-002 |
| **Priority** | P2 |
| **SRS Trace** | FR-OPS-002 |

**Test Steps:**
1. GET `/metrics`.
2. Assert HTTP 200.
3. Assert `Content-Type: text/plain`.
4. Assert body contains `apcore_a2a_tasks_total`.

**Expected Result:** Prometheus text format with `apcore_a2a_tasks_total` metric.

---

## 9. Integration Tests

---

#### TC-INT-001: Full message/send flow — client to server to executor to response

| Field | Value |
|-------|-------|
| **ID** | TC-INT-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-001, FR-EXE-001, FR-TSK-002 |

**Preconditions:** ASGI test server with 1 module `math.add` (executor returns `{"sum": inputs["a"] + inputs["b"]}`).

**Test Steps:**
1. POST JSON-RPC `message/send`:
   ```json
   {
     "jsonrpc": "2.0", "id": "req-1", "method": "message/send",
     "params": {
       "message": {
         "role": "user", "messageId": "msg-uuid-001",
         "parts": [{"kind": "data", "data": {"a": 3, "b": 4}}],
         "metadata": {"skillId": "math.add"}
       }
     }
   }
   ```
2. Assert HTTP 200.
3. Assert `result.kind == "task"`.
4. Assert `result.status.state == "completed"`.
5. Assert `result.artifacts[0].parts[0].data == {"sum": 7}`.

**Expected Result:** Completed task with `{"sum": 7}` artifact.

---

#### TC-INT-002: Full streaming flow — SSE events received in correct order

| Field | Value |
|-------|-------|
| **ID** | TC-INT-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-002, FR-TSK-002 |

**Preconditions:** Module `data.count` streams 3 chunks: `{"n": 1}`, `{"n": 2}`, `{"n": 3}`.

**Test Steps:**
1. POST `message/stream` for `data.count`.
2. Collect all SSE events.
3. Assert first event: `kind == "status-update"`, `state == "submitted"`.
4. Assert 3 artifact-update events follow.
5. Assert last event: `kind == "status-update"`, `state == "completed"`, `final == True`.

**Expected Result:** Events in order: submitted → 3×artifact → completed.

---

#### TC-INT-003: Multi-turn conversation — contextId preserved across tasks

| Field | Value |
|-------|-------|
| **ID** | TC-INT-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-CTX-001, FR-CTX-002 |

**Test Steps:**
1. Send first message with no `contextId` — record returned `task.contextId` as `ctx-abc`.
2. Send second message with `contextId = "ctx-abc"`.
3. Assert second task `contextId == "ctx-abc"`.
4. Call `tasks/list` filtering by `contextId == "ctx-abc"`.
5. Assert 2 tasks returned.

**Expected Result:** Both tasks share same contextId; list returns 2 tasks.

---

#### TC-INT-004: Authenticated message flow — JWT identity flows to ACL

| Field | Value |
|-------|-------|
| **ID** | TC-INT-004 |
| **Priority** | P1 |
| **SRS Trace** | FR-AUT-003, FR-AUT-004 |

**Preconditions:** Server with JWT auth; executor checks identity and raises `ACLDeniedError` for non-admin users.

**Test Steps:**
1. POST `message/send` with admin JWT (`roles: ["admin"]`). Assert task completed.
2. POST `message/send` with viewer JWT (`roles: ["viewer"]`). Assert `error.code == -32001`.

**Expected Result:** Admin succeeds, viewer gets ACL error.

---

#### TC-INT-005: Error propagation — executor error maps to A2A error

| Field | Value |
|-------|-------|
| **ID** | TC-INT-005 |
| **Priority** | P0 |
| **SRS Trace** | FR-ERR-001, FR-TSK-002 |

**Preconditions:** Executor raises `ModuleExecuteError("internal failure")` for `fail.module`.

**Test Steps:**
1. POST `message/send` for `fail.module`.
2. Assert `result.status.state == "failed"`.
3. Assert `result.status.message` is non-empty but does NOT contain "internal failure" (sanitized).

**Expected Result:** Failed task with sanitized error message.

---

#### TC-INT-006: A2A Client calls server — discover and send

| Field | Value |
|-------|-------|
| **ID** | TC-INT-006 |
| **Priority** | P0 |
| **SRS Trace** | FR-CLI-001, FR-CLI-002 |

**Preconditions:** Real ASGI test server running at `http://testserver`.

**Test Steps:**
1. Create `A2AClient(base_url="http://testserver", http_client=test_client)`.
2. `card = await client.discover()` — assert card has `name` and `skills`.
3. `task = await client.send_message(skill_id="math.add", inputs={"a": 10, "b": 5})`.
4. Assert `task.status.state == "completed"`.

**Expected Result:** Discovery and task execution both succeed.

---

#### TC-INT-007: input_required state — requires_approval flow

| Field | Value |
|-------|-------|
| **ID** | TC-INT-007 |
| **Priority** | P1 |
| **SRS Trace** | FR-CTX-003 |

**Preconditions:** Module with `requires_approval = True` — raises `ApprovalPendingError` on first call.

**Test Steps:**
1. Send `message/send` for approval-required module.
2. Assert task state is `input_required`.
3. Send follow-up `message/send` with same `contextId` and approval confirmation.
4. Assert task transitions to `completed`.

**Expected Result:** Task pauses at input_required, resumes after approval.

---

#### TC-INT-008: tasks/list with context_id filter returns correct tasks

| Field | Value |
|-------|-------|
| **ID** | TC-INT-008 |
| **Priority** | P0 |
| **SRS Trace** | FR-TSK-005 |

**Test Steps:**
1. Create 3 tasks with `context_id = "ctx-X"`.
2. Create 2 tasks with `context_id = "ctx-Y"`.
3. POST `tasks/list` with `params = {"contextId": "ctx-X"}`.
4. Assert 3 tasks returned.
5. POST `tasks/list` with `params = {"contextId": "ctx-Y"}`.
6. Assert 2 tasks returned.

**Expected Result:** Filtering by contextId works correctly.

---

## 10. End-to-End Tests

---

#### TC-E2E-001: Full server start, discover, send, verify response

| Field | Value |
|-------|-------|
| **ID** | TC-E2E-001 |
| **Priority** | P0 |
| **SRS Trace** | FR-SRV-001, FR-AGC-003, FR-MSG-001 |

**Test Steps:**
1. Start real uvicorn server on port 19001 in subprocess.
2. `GET http://localhost:19001/.well-known/agent.json` — assert 200.
3. POST `message/send` to `http://localhost:19001/` — assert completed task.
4. Stop server.

**Expected Result:** Full lifecycle from server start to task completion.

---

#### TC-E2E-002: Full streaming E2E — receive all SSE events

| Field | Value |
|-------|-------|
| **ID** | TC-E2E-002 |
| **Priority** | P0 |
| **SRS Trace** | FR-MSG-002 |

**Test Steps:**
1. Start real server with streaming module.
2. Open SSE connection via `httpx` streaming GET.
3. Collect all events until stream closes.
4. Assert events contain: submitted → working → ≥1 artifact → completed.
5. Assert stream closed after completed event.

**Expected Result:** SSE stream with all expected events, cleanly closed.

---

#### TC-E2E-003: Two-agent interaction — server A calls server B

| Field | Value |
|-------|-------|
| **ID** | TC-E2E-003 |
| **Priority** | P0 |
| **SRS Trace** | FR-CLI-001, FR-CLI-002, FR-SRV-001 |

**Test Steps:**
1. Start server A at port 19002 with module `echo.text`.
2. Start server B at port 19003 — B has a module that internally uses `A2AClient` to call A.
3. Send message to server B triggering A→B chain.
4. Assert final result contains echo from server A.

**Expected Result:** Cross-agent task delegation works end-to-end.

---

#### TC-E2E-004: Authenticated E2E — JWT required, invalid rejected

| Field | Value |
|-------|-------|
| **ID** | TC-E2E-004 |
| **Priority** | P1 |
| **SRS Trace** | FR-AUT-002 |

**Test Steps:**
1. Start server with `JWTAuthenticator(secret="e2e-secret-xyz")` and `require_auth=True`.
2. POST `message/send` without Authorization. Assert HTTP 401.
3. POST with `Authorization: Bearer {valid_jwt}`. Assert HTTP 200, task completed.
4. POST with `Authorization: Bearer {expired_jwt}`. Assert HTTP 401.

**Expected Result:** Valid JWT succeeds, missing/expired rejected.

---

## 11. Performance Tests

---

#### TC-PERF-001: Agent Card generation < 50ms for 100 modules

| Field | Value |
|-------|-------|
| **ID** | TC-PERF-001 |
| **Priority** | P0 |
| **SRS Trace** | NFR-PERF-001 |

**Test Steps:**
1. Create registry with 100 stub modules.
2. Time `AgentCardBuilder(registry, url="http://localhost").build()` 10 times.
3. Assert p99 < 50ms.

**Expected Result:** P99 < 50ms.

---

#### TC-PERF-002: message/send latency (excluding executor) < 10ms overhead

| Field | Value |
|-------|-------|
| **ID** | TC-PERF-002 |
| **Priority** | P0 |
| **SRS Trace** | NFR-PERF-002 |

**Preconditions:** Stub executor that returns immediately (0ms).

**Test Steps:**
1. Send 100 `message/send` requests sequentially.
2. Record latency for each.
3. Assert p95 < 10ms (overhead from routing, not executor).

**Expected Result:** P95 overhead < 10ms.

---

#### TC-PERF-003: 100 concurrent tasks without errors

| Field | Value |
|-------|-------|
| **ID** | TC-PERF-003 |
| **Priority** | P0 |
| **SRS Trace** | NFR-SCA-001 |

**Test Steps:**
1. Launch 100 concurrent `message/send` requests using `asyncio.gather()`.
2. Assert all 100 complete without error.
3. Assert all 100 tasks reach `completed` state.
4. Total time < 5 seconds.

**Expected Result:** 100 concurrent tasks all completed.

---

#### TC-PERF-004: SSE streaming throughput — 10 chunks per second sustained

| Field | Value |
|-------|-------|
| **ID** | TC-PERF-004 |
| **Priority** | P1 |
| **SRS Trace** | NFR-PERF-003 |

**Preconditions:** Module produces 50 chunks with 100ms delay between each.

**Test Steps:**
1. Start streaming and measure event receipt timestamps.
2. Assert all 50 events received within `50 * 100ms + 2000ms` tolerance.
3. Assert no event lost.

**Expected Result:** All chunks delivered, no loss.

---

#### TC-PERF-005: InMemoryTaskStore handles 1000 concurrent operations

| Field | Value |
|-------|-------|
| **ID** | TC-PERF-005 |
| **Priority** | P0 |
| **SRS Trace** | NFR-SCA-002 |

**Test Steps:**
1. Create `InMemoryTaskStore()`.
2. Concurrently execute: 400 save, 400 get, 200 list operations.
3. Assert all operations complete without exception.
4. Total time < 2 seconds.

**Expected Result:** No errors, all operations complete.

---

## 12. Security Tests

---

#### TC-SEC-001: Unauthenticated request rejected when require_auth=True

| Field | Value |
|-------|-------|
| **ID** | TC-SEC-001 |
| **Priority** | P0 |
| **SRS Trace** | NFR-SEC-001 |

**Test Steps:**
1. Server with `require_auth=True`.
2. POST `message/send` with no Authorization header.
3. Assert HTTP 401.
4. Assert response body does NOT contain any module names or internal paths.

**Expected Result:** HTTP 401, no internal info leaked.

---

#### TC-SEC-002: Expired JWT token rejected with 401

| Field | Value |
|-------|-------|
| **ID** | TC-SEC-002 |
| **Priority** | P0 |
| **SRS Trace** | NFR-SEC-001, FR-AUT-001 |

**Test Steps:**
1. Generate JWT expired 1 hour ago.
2. POST with `Authorization: Bearer {expired}`.
3. Assert HTTP 401.

**Expected Result:** HTTP 401.

---

#### TC-SEC-003: ACL error message does not leak caller or module info

| Field | Value |
|-------|-------|
| **ID** | TC-SEC-003 |
| **Priority** | P0 |
| **SRS Trace** | NFR-SEC-003, FR-ERR-001 |

**Preconditions:** Executor raises `ACLDeniedError(caller="service-account-db-writer", module="admin.users.delete")`.

**Test Steps:**
1. POST `message/send` for `admin.users.delete` with insufficient permissions.
2. Parse response — assert `result.status.state == "failed"` or `error.code == -32001`.
3. Assert `"service-account-db-writer"` NOT in response body.
4. Assert `"admin.users.delete"` NOT in response body.

**Expected Result:** Error returned with no sensitive info.

---

#### TC-SEC-004: Internal error details not exposed to client

| Field | Value |
|-------|-------|
| **ID** | TC-SEC-004 |
| **Priority** | P0 |
| **SRS Trace** | NFR-SEC-003 |

**Preconditions:** Executor raises `RuntimeError("Database password: P@ssw0rd123")`.

**Test Steps:**
1. POST `message/send` triggering the error.
2. Assert `"P@ssw0rd123"` NOT in any field of the response.
3. Assert response has generic error message.

**Expected Result:** Password not in response.

---

#### TC-SEC-005: Push notification token validated on delivery

| Field | Value |
|-------|-------|
| **ID** | TC-SEC-005 |
| **Priority** | P1 |
| **SRS Trace** | NFR-SEC-004 |

**Test Steps:**
1. Register push config with `token = "my-verify-token-abc"`.
2. Receive webhook POST at mock server.
3. Assert `X-A2A-Notification-Token: my-verify-token-abc` header present.
4. Simulate client-side: reject webhook if token does not match. Assert security hold enforced.

**Expected Result:** Token delivered for client-side validation.

---

#### TC-SEC-006: Request body > 10MB returns 413 (DoS protection)

| Field | Value |
|-------|-------|
| **ID** | TC-SEC-006 |
| **Priority** | P0 |
| **SRS Trace** | NFR-SEC-005 |

**Test Steps:**
1. POST `/` with body of 11 * 1024 * 1024 bytes.
2. Assert HTTP 413.

**Expected Result:** HTTP 413.

---

## 13. Test Data and Fixtures

### 13.1 Stub Module Descriptors

```python
# Simple module
SIMPLE_DESCRIPTOR = StubDescriptor(
    module_id="math.add",
    description="Adds two integers and returns their sum",
    tags=["math", "arithmetic"],
    input_schema={"type": "object", "properties": {"a": {"type": "integer"}, "b": {"type": "integer"}}, "required": ["a", "b"]},
    output_schema={"type": "object", "properties": {"sum": {"type": "integer"}}},
    examples=[StubExample(title="Add 3 and 4", inputs={"a": 3, "b": 4})],
    annotations=StubAnnotations(readonly=True, idempotent=True)
)

# Streaming module
STREAMING_DESCRIPTOR = StubDescriptor(
    module_id="data.stream",
    description="Streams a sequence of numbers",
    tags=["data", "streaming"],
    input_schema={"type": "object", "properties": {"count": {"type": "integer"}}},
    output_schema={"type": "object", "properties": {"n": {"type": "integer"}}}
)

# Destructive module
DESTRUCTIVE_DESCRIPTOR = StubDescriptor(
    module_id="file.delete",
    description="Deletes a file at the given path",
    tags=["file", "delete"],
    input_schema={"type": "object", "properties": {"path": {"type": "string"}}},
    annotations=StubAnnotations(destructive=True, requires_approval=True)
)

# Module with nested schema ($defs)
NESTED_SCHEMA_DESCRIPTOR = StubDescriptor(
    module_id="image.resize",
    description="Resizes an image to specified dimensions",
    tags=["image", "resize"],
    input_schema={
        "type": "object",
        "$defs": {"Dimensions": {"type": "object", "properties": {"width": {"type": "integer"}, "height": {"type": "integer"}}}},
        "properties": {"size": {"$ref": "#/$defs/Dimensions"}}
    }
)

# Module with empty description (should be excluded)
NO_DESCRIPTION_DESCRIPTOR = StubDescriptor(
    module_id="internal.debug",
    description="",
    tags=[]
)
```

### 13.2 JWT Tokens

```python
import jwt
import time

SECRET = "test-secret-key-abc123"

# Valid admin token
VALID_ADMIN_JWT = jwt.encode(
    {"sub": "admin-user-001", "type": "user", "roles": ["admin"], "exp": int(time.time()) + 3600},
    SECRET, algorithm="HS256"
)

# Valid viewer token
VALID_VIEWER_JWT = jwt.encode(
    {"sub": "viewer-user-002", "type": "user", "roles": ["viewer"], "exp": int(time.time()) + 3600},
    SECRET, algorithm="HS256"
)

# Expired token
EXPIRED_JWT = jwt.encode(
    {"sub": "expired-user-003", "exp": int(time.time()) - 3600},
    SECRET, algorithm="HS256"
)

# Wrong signature
WRONG_SIG_JWT = jwt.encode(
    {"sub": "bad-user-004", "exp": int(time.time()) + 3600},
    "wrong-secret", algorithm="HS256"
)
```

### 13.3 Sample JSON-RPC Requests

```python
# message/send request
MESSAGE_SEND_REQUEST = {
    "jsonrpc": "2.0",
    "id": "req-001",
    "method": "message/send",
    "params": {
        "message": {
            "role": "user",
            "messageId": "msg-uuid-001",
            "parts": [{"kind": "data", "data": {"a": 3, "b": 4}}],
            "metadata": {"skillId": "math.add"}
        }
    }
}

# tasks/get request
TASKS_GET_REQUEST = lambda task_id: {
    "jsonrpc": "2.0",
    "id": "req-002",
    "method": "tasks/get",
    "params": {"id": task_id, "historyLength": 10}
}

# tasks/cancel request
TASKS_CANCEL_REQUEST = lambda task_id: {
    "jsonrpc": "2.0",
    "id": "req-003",
    "method": "tasks/cancel",
    "params": {"id": task_id}
}
```

### 13.4 Sample Agent Card

```json
{
  "name": "Test Agent",
  "description": "A test apcore agent with 2 skills",
  "version": "1.0.0",
  "url": "http://localhost:8000",
  "defaultInputModes": ["application/json", "text/plain"],
  "defaultOutputModes": ["application/json"],
  "capabilities": {
    "streaming": true,
    "pushNotifications": false,
    "stateTransitionHistory": false
  },
  "skills": [
    {
      "id": "math.add",
      "description": "Adds two integers and returns their sum",
      "tags": ["math", "arithmetic"],
      "examples": [{"name": "Add 3 and 4", "input": "{\"a\": 3, \"b\": 4}"}],
      "inputModes": ["application/json"],
      "outputModes": ["application/json"],
      "extensions": {"apcore": {"annotations": {"readonly": true, "destructive": false, "idempotent": true, "requires_approval": false, "open_world": true}}}
    }
  ]
}
```

---

## 14. Traceability Matrix

### 14.1 Functional Requirements → Test Cases

| SRS Requirement | Description | Test Cases |
|----------------|-------------|------------|
| FR-SRV-001 | serve() launches A2A server | TC-SRV-001, TC-SRV-002, TC-E2E-001 |
| FR-SRV-002 | serve() keyword arguments | TC-SRV-001, TC-SRV-003 |
| FR-SRV-003 | async_serve() returns ASGI app | TC-SRV-003 |
| FR-SRV-004 | JSON-RPC 2.0 dispatch | TC-SRV-004, TC-SRV-005, TC-SRV-006, TC-SRV-007 |
| FR-SRV-005 | Graceful shutdown | TC-SRV-008 (future) |
| FR-AGC-001 | Agent Card from Registry metadata | TC-AGC-001, TC-AGC-002 |
| FR-AGC-002 | Capabilities computed | TC-AGC-003, TC-AGC-004 |
| FR-AGC-003 | Serve Agent Card at /.well-known/agent.json | TC-AGC-005, TC-AGC-006 |
| FR-AGC-004 | Extended Agent Card | TC-AGC-007 |
| FR-AGC-005 | Regenerate Agent Card on changes | TC-AGC-008 |
| FR-SKL-001 | Module → Skill mapping | TC-SKL-001, TC-SKL-002, TC-SKL-003, TC-SKL-004, TC-SKL-005 |
| FR-SKL-002 | Examples mapping | TC-SKL-006 |
| FR-SKL-003 | Input/output modes | TC-SKL-007, TC-SKL-008 |
| FR-SKL-004 | Annotations as extensions | TC-SKL-009, TC-SKL-010 |
| FR-MSG-001 | message/send synchronous | TC-INT-001, TC-INT-005 |
| FR-MSG-002 | message/stream SSE | TC-STR-001–006, TC-INT-002, TC-E2E-002 |
| FR-MSG-003 | Parse Parts to module input | TC-PRT-003–006 |
| FR-MSG-004 | Convert output to Artifacts | TC-PRT-001, TC-PRT-002 |
| FR-TSK-001 | Task created with submitted state | TC-TSK-001, TC-TSK-009 |
| FR-TSK-002 | Valid state transitions | TC-TSK-002, TC-INT-001 |
| FR-TSK-003 | Invalid transitions rejected | TC-TSK-003, TC-TSK-004 |
| FR-TSK-005 | tasks/get | TC-TSK-005, TC-TSK-006 |
| FR-TSK-006 | tasks/cancel | TC-TSK-007, TC-TSK-008 |
| FR-TSK-007 | tasks/resubscribe | TC-STR-005 |
| FR-EXE-001 | Execution routing | TC-RTR-001–005 |
| FR-EXE-003 | Schema converter | TC-SCH-001–005 |
| FR-ERR-001 | Error mapping | TC-ERR-001–005 |
| FR-CLI-001 | Client agent discovery | TC-CLI-001, TC-CLI-002, TC-INT-006 |
| FR-CLI-002 | Client send_message | TC-CLI-003, TC-CLI-006, TC-INT-006 |
| FR-CLI-003 | Client stream_message | TC-CLI-004 |
| FR-CLI-004 | Client get_task | TC-CLI-005 |
| FR-CTX-001 | contextId grouping | TC-INT-003, TC-INT-008 |
| FR-CTX-003 | input_required state | TC-INT-007 |
| FR-PSH-001 | Push notification config CRUD | TC-PSH-001, TC-PSH-005 |
| FR-PSH-002 | Push notification delivery | TC-PSH-002, TC-PSH-003 |
| FR-PSH-003 | Push notification retry | TC-PSH-004 |
| FR-AUT-001 | JWT validation | TC-AUT-001–004 |
| FR-AUT-002 | AuthMiddleware | TC-AUT-005, TC-AUT-006 |
| FR-AUT-004 | Identity bridging | TC-RTR-005, TC-INT-004 |
| FR-STR-001 | TaskStore CRUD | TC-STO-001, TC-STO-002, TC-STO-004 |
| FR-STR-002 | Concurrent access | TC-STO-003 |
| FR-CMD-001 | CLI entry point | TC-CMD-001–003 |
| FR-EXP-001 | Explorer UI | TC-EXP-001–003 |
| FR-OPS-001 | Health endpoint | TC-OPS-001 |
| FR-OPS-002 | Metrics endpoint | TC-OPS-002 |

### 14.2 Non-Functional Requirements → Test Cases

| SRS Requirement | Description | Test Cases |
|----------------|-------------|------------|
| NFR-PERF-001 | Agent Card generation < 50ms | TC-PERF-001 |
| NFR-PERF-002 | message/send latency overhead < 10ms | TC-PERF-002 |
| NFR-PERF-003 | SSE streaming throughput | TC-PERF-004 |
| NFR-SEC-001 | Auth enforcement | TC-SEC-001, TC-SEC-002, TC-E2E-004 |
| NFR-SEC-003 | Error sanitization | TC-ERR-003, TC-ERR-004, TC-SEC-003, TC-SEC-004 |
| NFR-SEC-004 | Push notification token | TC-PSH-003, TC-SEC-005 |
| NFR-SEC-005 | Request size limit | TC-SRV-007, TC-SEC-006 |
| NFR-REL-003 | Concurrent access safety | TC-STO-003, TC-TSK-009 |
| NFR-SCA-001 | 100 concurrent tasks | TC-PERF-003 |
| NFR-SCA-002 | Storage under load | TC-PERF-005 |
| NFR-MNT-001 | 90% code coverage | Enforced via pytest-cov |

---

## 15. Quality Gates

### 15.1 Entry Criteria (Tests can begin)

- Component implementation matches tech design interfaces
- All stub fixtures defined in `tests/conftest.py`
- `ruff check src/ tests/` passes with zero violations
- `mypy src/` passes with zero errors

### 15.2 Exit Criteria (Release can proceed)

| Gate | Requirement | Measurement |
|------|-------------|-------------|
| Coverage | Line coverage ≥ 90%, branch ≥ 85% | `pytest --cov --cov-fail-under=90` |
| P0 tests | All P0 test cases passing | 0 failures in P0 suite |
| P1 tests | ≤ 5% P1 failures allowed | < 3 P1 failures |
| Performance | TC-PERF-001 and TC-PERF-003 pass | Latency and concurrency targets met |
| Security | All TC-SEC-* pass | 0 security test failures |
| Linting | ruff + mypy clean | 0 violations |
| E2E | All TC-E2E-* pass | 0 E2E failures |

### 15.3 Test Count Summary

| Category | Count |
|----------|-------|
| TC-AGC (Agent Card) | 8 |
| TC-SKL (Skill Mapper) | 10 |
| TC-SCH (Schema Converter) | 5 |
| TC-PRT (Part Converter) | 6 |
| TC-ERR (Error Mapper) | 5 |
| TC-SRV (Server Factory) | 7 |
| TC-RTR (Execution Router) | 5 |
| TC-TSK (Task Manager) | 9 |
| TC-STR (Streaming Handler) | 6 |
| TC-PSH (Push Notification) | 5 |
| TC-CLI (A2A Client) | 6 |
| TC-AUT (Authentication) | 6 |
| TC-STO (Storage) | 4 |
| TC-CMD (CLI) | 3 |
| TC-EXP (Explorer) | 3 |
| TC-OPS (Operations) | 2 |
| TC-INT (Integration) | 8 |
| TC-E2E (End-to-End) | 4 |
| TC-PERF (Performance) | 5 |
| TC-SEC (Security) | 6 |
| **Total** | **118** |

---

## 16. Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| a2a-sdk API changes during development | Medium | High | Pin SDK version; add integration tests against SDK boundary |
| SSE streaming race conditions | Medium | High | Use asyncio locks; TC-STR tests cover concurrent scenarios |
| InMemoryTaskStore corruption under high concurrency | Low | High | TC-STO-003 stress test; use asyncio.Lock per task |
| Push notification delivery failures in tests | Medium | Medium | Use mock HTTP server; TC-PSH-004 covers retry logic |
| apcore-python API changes breaking adapter | Low | High | Pin version in dev; stub all apcore calls in unit tests |
| Test isolation failures (shared state between tests) | Low | Medium | Use fresh fixtures per test; pytest-asyncio in auto mode |
| CI flakiness in performance tests | Medium | Low | Run perf tests separately; use conservative thresholds |
