# Software Requirements Specification: apcore-a2a

| Field       | Value                                                                    |
|-------------|--------------------------------------------------------------------------|
| Title       | apcore-a2a: Automatic A2A Protocol Adapter for apcore Module Registry    |
| Document    | Software Requirements Specification (SRS)                                |
| Document ID | SRS-APCORE-A2A-001                                                       |
| Version     | 1.0                                                                      |
| Date        | 2026-03-03                                                               |
| Author      | aipartnerup Engineering Team                                             |
| Status      | Draft                                                                    |
| PRD Ref     | `docs/prd.md` v1.0                                                      |
| Standard    | IEEE 830 / ISO/IEC/IEEE 29148                                            |

---

## Revision History

| Version | Date       | Author                      | Description         |
|---------|------------|------------------------------|---------------------|
| 1.0     | 2026-03-03 | aipartnerup Engineering Team | Initial draft       |

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Overall Description](#2-overall-description)
3. [Specific Requirements -- Functional Requirements](#3-specific-requirements----functional-requirements)
4. [Specific Requirements -- Non-Functional Requirements](#4-specific-requirements----non-functional-requirements)
5. [Use Cases](#5-use-cases)
6. [CRUD Matrix](#6-crud-matrix)
7. [Data Requirements](#7-data-requirements)
8. [Interface Requirements](#8-interface-requirements)
9. [Traceability Matrix](#9-traceability-matrix)
10. [Appendices](#10-appendices)

---

## 1. Introduction

### 1.1 Purpose

This Software Requirements Specification defines the complete functional and non-functional requirements for **apcore-a2a**, an independent Python adapter package that automatically bridges any apcore Module Registry into a fully functional A2A (Agent-to-Agent) protocol compliant agent server and client. This document formalizes all 19 features from the upstream PRD (FR-001 through FR-019) into traceable, testable requirements organized by component module. It serves as the authoritative reference for implementation, testing, and acceptance of apcore-a2a v1.0.0.

The intended audience includes software engineers implementing apcore-a2a, QA engineers writing test plans, and project stakeholders evaluating feature completeness.

### 1.2 Scope

apcore-a2a provides two primary capabilities:

1. **A2A Server Bridge**: A `serve(registry)` function that launches a standards-compliant A2A agent server exposing all apcore modules as A2A Skills, with automatic Agent Card generation, full task lifecycle management, SSE streaming, and push notification support.
2. **A2A Client**: An `A2AClient` class for discovering and invoking remote A2A-compliant agents, enabling apcore agents to participate in multi-agent workflows as both servers and clients.

The system reads apcore module metadata (`input_schema`, `output_schema`, `description`, `annotations`, `tags`, `examples`) and generates a complete A2A agent at runtime with zero per-module code. All task executions are routed through the apcore `Executor` pipeline, preserving ACL enforcement, input validation, middleware execution, and timeout enforcement.

apcore-a2a does NOT reimplement the A2A protocol (it uses the official `a2a-sdk` Python package), does NOT define modules (that is apcore-python's responsibility), does NOT implement domain-specific wrapping (that is the job of `xxx-apcore` projects), and does NOT implement gRPC bindings (JSON-RPC + HTTP first).

### 1.3 Definitions, Acronyms, and Abbreviations

| Term | Definition |
|------|------------|
| **A2A** | Agent-to-Agent protocol. Open standard by Google/Linux Foundation for inter-agent communication, discovery, and task delegation. |
| **Agent Card** | JSON document served at `/.well-known/agent.json` describing an A2A agent's identity, capabilities, skills, and security schemes. |
| **apcore** | Schema-driven module development framework providing Registry, Executor, and schema validation for modular applications. |
| **apcore-python** | The Python SDK implementation of the apcore framework. Provides `Registry`, `Executor`, `ModuleDescriptor`, `ModuleAnnotations`, and the error hierarchy. |
| **Artifact** | Output produced by an A2A task. Contains Parts (TextPart, DataPart, FilePart) representing the result of module execution. |
| **ASGI** | Asynchronous Server Gateway Interface. Python standard for async web servers and applications. |
| **contextId** | UUID v4 identifier grouping multiple messages into a multi-turn conversation. |
| **Executor** | apcore component that orchestrates the module execution pipeline: context creation, safety checks, ACL enforcement, input validation, middleware, execution, and result return. |
| **JSON-RPC 2.0** | Lightweight remote procedure call protocol using JSON encoding. Primary protocol binding for A2A communication. |
| **MCP** | Model Context Protocol. Anthropic's protocol for model-to-tool communication. Complementary to A2A. |
| **Module** | An apcore capability unit with defined `input_schema`, `output_schema`, `description`, `annotations`, `tags`, and `examples`. |
| **ModuleAnnotations** | apcore frozen dataclass with behavioral flags: `readonly`, `destructive`, `idempotent`, `requires_approval`, `open_world`. |
| **ModuleDescriptor** | apcore dataclass containing a module's full metadata: `module_id`, `name`, `description`, `input_schema`, `output_schema`, `annotations`, `examples`, `tags`, `version`. |
| **Part** | A piece of content within an A2A Message or Artifact. Types: TextPart, DataPart, FilePart. |
| **Push Notification** | Webhook-based delivery of task state updates via HTTP POST. |
| **Registry** | apcore component that discovers, loads, and indexes modules. Provides `list()`, `get_definition()`, and event-based registration notifications. |
| **Skill** | An A2A concept representing a specific capability of an agent. Maps 1:1 to apcore modules. |
| **SSE** | Server-Sent Events. HTTP-based unidirectional streaming used for A2A `message/stream`. |
| **Task** | Stateful A2A work unit with lifecycle: submitted, working, completed, failed, canceled, input_required. |
| **TaskStore** | Pluggable storage backend for persisting task state. |
| **xxx-apcore** | Convention for apcore adapter projects targeting specific domains (e.g., `comfyui-apcore`). |

### 1.4 References

| ID   | Reference | Description |
|------|-----------|-------------|
| REF-01 | `docs/apcore-a2a/prd.md` v1.0 | Product Requirements Document -- primary input for this SRS |
| REF-02 | `ideas/apcore-a2a.md` | Validated idea document with schema mapping and architecture context |
| REF-03 | apcore-python source (`apcore` package >= 0.6.0) | SDK source: Registry, Executor, ModuleDescriptor, ModuleAnnotations, error hierarchy |
| REF-04 | [A2A Protocol Specification (v0.3.0)](https://google.github.io/A2A/) | Official Agent-to-Agent protocol specification |
| REF-05 | [A2A Python SDK (a2a-sdk)](https://pypi.org/project/a2a-sdk/) | Official Python SDK for building A2A agents and clients |
| REF-06 | apcore-mcp SRS (`docs/srs-apcore-mcp.md`) | Sibling adapter SRS -- architectural pattern reference |
| REF-07 | IEEE 830-1998 | IEEE Recommended Practice for SRS |
| REF-08 | ISO/IEC/IEEE 29148:2018 | Systems and software engineering -- Life cycle processes -- Requirements engineering |
| REF-09 | [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification) | JSON-RPC 2.0 protocol specification |

### 1.5 Overview

Section 2 provides the overall product description, including ecosystem context, user characteristics, constraints, and assumptions. Section 3 specifies all functional requirements organized by component module (15 module groups, 85 requirements). Section 4 specifies non-functional requirements for performance, security, reliability, compatibility, maintainability, and scalability. Section 5 defines 6 detailed use cases for major workflows. Sections 6-8 provide the CRUD matrix, data requirements, and interface requirements. Section 9 contains the bidirectional traceability matrix. Section 10 is the appendix with glossary and protocol references.

---

## 2. Overall Description

### 2.1 Product Perspective

apcore-a2a is a thin adapter layer that sits between the apcore-python SDK and the A2A agent ecosystem. It occupies a specific architectural position defined by apcore's SCOPE.md:

```
apcore-python (core SDK)
    |
    +-- Provides: Registry, Executor, ModuleDescriptor, ModuleAnnotations, error hierarchy
    |
apcore-mcp (sibling adapter -- MCP protocol)
    |
    +-- Produces: MCP Server, OpenAI tools list
    +-- Targets: AI assistants (Claude Desktop, Cursor, GPT)
    |
apcore-a2a (this project -- A2A protocol)
    |
    +-- Consumes: Registry API, Executor.call_async() / stream()
    +-- Produces: A2A Server (Agent Card, JSON-RPC, SSE), A2A Client
    |
    +-- Server path --> Other A2A agents, orchestrators, enterprise systems
    +-- Client path --> Remote A2A agents (any framework)
```

apcore-a2a is the second adapter in the planned family. It depends on the `apcore` package for module metadata and execution, and on the `a2a-sdk` package for A2A protocol types and utilities. Combined with apcore-mcp, it establishes apcore as the only schema-driven framework with native dual-protocol coverage (MCP + A2A) from a single module definition.

### 2.2 Product Functions (High-Level Summary)

1. **Agent Card Generation**: Construct A2A Agent Card from apcore Registry metadata (name, description, version, skills, capabilities).
2. **Skill Mapping**: Convert apcore module definitions to A2A Skill objects with accurate metadata preservation.
3. **A2A Server Launch**: Create and start an A2A HTTP server via `serve()` with Agent Card endpoint and JSON-RPC handling.
4. **Synchronous Execution**: Route `message/send` requests through the apcore Executor pipeline and return completed Tasks.
5. **SSE Streaming**: Route `message/stream` requests through Executor streaming pipeline with real-time TaskStatusUpdateEvent and TaskArtifactUpdateEvent delivery.
6. **Task Lifecycle Management**: Implement the A2A task state machine (submitted, working, completed, failed, canceled, input_required) with thread-safe transitions.
7. **Execution Routing**: Bridge A2A message Parts to apcore Executor calls, preserving ACL, validation, and middleware.
8. **Error Mapping**: Translate apcore error hierarchy to A2A JSON-RPC error codes with security-aware sanitization.
9. **Multi-Turn Conversations**: Group messages by `contextId` for stateful, multi-turn interactions with input elicitation.
10. **A2A Client**: Discover and invoke remote A2A agents via `A2AClient` class.
11. **Push Notifications**: Webhook-based async task state delivery with retry logic.
12. **Authentication Bridging**: JWT/Bearer token validation bridged to apcore Identity context for ACL enforcement.
13. **Task Storage**: Pluggable task store interface with default in-memory implementation.
14. **CLI Entry Point**: Command-line server launch without writing Python code.
15. **Explorer UI**: Optional browser-based skill exploration and message testing.
16. **Operations**: Health check and metrics endpoints for production monitoring.
17. **Dynamic Registration**: Runtime module addition/removal with automatic Agent Card updates.

### 2.3 User Characteristics

| Persona | Role | Experience | Primary Interaction |
|---------|------|------------|---------------------|
| **Maya** (Module Developer) | Writes apcore modules, wants A2A agent integration | 3-5 years Python, familiar with apcore-python | Calls `serve(registry)` to expose modules as A2A agent |
| **Omar** (Agent Orchestrator) | Builds multi-agent workflows coordinating diverse AI agents | 5+ years, familiar with A2A protocol and orchestration | Discovers and invokes apcore agents via standard A2A protocol |
| **Priya** (xxx-apcore Project Dev) | Maintains domain-specific apcore bridges | Moderate Python, familiar with apcore and domain tool | Adds `apcore-a2a` dependency, calls `serve()` |
| **Elena** (Enterprise Architect) | Designs secure multi-agent systems for enterprise | 10+ years, evaluates for security/compliance/scalability | Configures JWT auth, reviews Agent Card security schemes |

### 2.4 Constraints

| ID | Constraint | Rationale |
|----|-----------|-----------|
| C-01 | Must use official `a2a-sdk` Python package for A2A protocol types and utilities | A2A protocol is complex; reimplementation is out of scope |
| C-02 | Must use apcore `Executor` for all task execution routing | ACL, validation, and middleware guarantees must be preserved |
| C-03 | Core logic must not exceed 1,500 lines (excluding tests, docs, and Explorer UI assets) | Thin adapter design principle |
| C-04 | Python >= 3.11 required | Aligns with apcore-python minimum; needed for modern async features |
| C-05 | Authentication bridges to apcore ACL, not replaces it | JWT validation at HTTP layer injects Identity into Context for Executor ACL |
| C-06 | A2A Client must be importable independently without server-side dependencies | Maximum flexibility; `from apcore_a2a.client import A2AClient` shall not import starlette/uvicorn |
| C-07 | Full compliance with A2A v0.3.0 specification | Interoperability with all A2A-compliant agents and clients |

### 2.5 Assumptions and Dependencies

| ID | Item | Type |
|----|------|------|
| A-01 | apcore-python SDK API (`Registry`, `Executor`, `ModuleDescriptor`, `ModuleAnnotations`) is stable and will not undergo breaking changes during development | Assumption |
| A-02 | The `a2a-sdk` Python package provides stable APIs for A2A types, JSON-RPC handling, and SSE utilities | Assumption |
| A-03 | apcore modules produce JSON-serializable output dicts from their `execute()` methods | Assumption |
| A-04 | Python >= 3.11 is available in the target deployment environment | Assumption |
| A-05 | A2A clients correctly implement the A2A v0.3.0 protocol for agent discovery and task operations | Assumption |
| D-01 | `apcore` package >= 0.6.0 | Dependency |
| D-02 | `a2a-sdk` package (latest stable) | Dependency |
| D-03 | `starlette` >= 0.27 | Dependency |
| D-04 | `uvicorn` >= 0.23 | Dependency |
| D-05 | `httpx` >= 0.24 | Dependency |
| D-06 | `PyJWT` >= 2.8 (optional, for JWT authentication) | Dependency |

---

## 3. Specific Requirements -- Functional Requirements

### 3.1 FR-SRV: Server Module Requirements

---

#### FR-SRV-001: serve() launches A2A HTTP server from Registry

| Field | Value |
|-------|-------|
| **ID** | FR-SRV-001 |
| **Priority** | P0 |
| **PRD Trace** | FR-001 |

**Description:** The system SHALL provide a top-level `serve(registry)` function that accepts an apcore `Registry` (or `Executor`) instance and starts a fully functional A2A-compliant HTTP server using uvicorn.

**Rationale:** This is the primary public API entry point, mirroring apcore-mcp's `serve(registry)` pattern. Developers must be able to expose all modules as A2A agents with a single function call.

**Acceptance Criteria:**
1. `serve(registry)` SHALL start an HTTP server on `0.0.0.0:8000` by default.
2. `serve(registry, host="127.0.0.1", port=9000)` SHALL bind to the specified host and port.
3. `serve(executor)` SHALL accept an Executor instance (duck-typed: any object with `call_async()`, `stream()`, `validate()` methods).
4. The server SHALL respond to A2A JSON-RPC 2.0 requests at the root path `/`.
5. The server SHALL respond to Agent Card requests at `GET /.well-known/agent.json`.
6. The server SHALL shut down cleanly on SIGINT and SIGTERM signals within 5 seconds.
7. The function SHALL block the calling thread until the server is shut down.
8. The function SHALL raise `ValueError` with a descriptive message if the registry/executor has zero modules.

---

#### FR-SRV-002: serve() accepts configuration keyword arguments

| Field | Value |
|-------|-------|
| **ID** | FR-SRV-002 |
| **Priority** | P0 |
| **PRD Trace** | FR-001 |

**Description:** The `serve()` function SHALL accept optional keyword arguments for customizing agent identity, authentication, storage, CORS, and push notification behavior.

**Rationale:** Different deployments require different configurations. The serve function must be customizable without requiring users to construct internal objects manually.

**Acceptance Criteria:**
1. `serve()` SHALL accept `name: str` (agent name; default: from Registry config or `"apcore-agent"`).
2. `serve()` SHALL accept `description: str` (agent description; default: auto-generated).
3. `serve()` SHALL accept `version: str` (agent version; default: from Registry config or `"0.0.0"`).
4. `serve()` SHALL accept `auth: Authenticator | None` (authentication middleware; default: `None`).
5. `serve()` SHALL accept `task_store: TaskStore | None` (task storage backend; default: `InMemoryTaskStore()`).
6. `serve()` SHALL accept `cors_origins: list[str] | None` (CORS allowed origins; default: `None` for same-origin only).
7. `serve()` SHALL accept `push_notifications: bool` (enable push notifications; default: `False`).
8. `serve()` SHALL accept `explorer: bool` (enable Explorer UI; default: `False`).

---

#### FR-SRV-003: async_serve() returns ASGI application without starting uvicorn

| Field | Value |
|-------|-------|
| **ID** | FR-SRV-003 |
| **Priority** | P0 |
| **PRD Trace** | FR-001 |

**Description:** The system SHALL provide an `async_serve()` variant that constructs and returns the ASGI application without starting uvicorn, enabling embedding in existing ASGI servers.

**Rationale:** Production deployments often embed applications within larger ASGI servers (hypercorn, daphne, or multi-app uvicorn). The ASGI app must be extractable.

**Acceptance Criteria:**
1. `async_serve(registry, **options)` SHALL return a Starlette/ASGI application instance.
2. The returned application SHALL be mountable in any ASGI-compliant server (uvicorn, hypercorn, daphne).
3. The returned application SHALL accept the same keyword arguments as `serve()` (excluding `host` and `port`).
4. The returned application SHALL NOT start a server process or bind to any network port.

---

#### FR-SRV-004: Server handles JSON-RPC 2.0 request dispatch

| Field | Value |
|-------|-------|
| **ID** | FR-SRV-004 |
| **Priority** | P0 |
| **PRD Trace** | FR-001 |

**Description:** The server SHALL accept HTTP POST requests at the root path `/` containing JSON-RPC 2.0 request objects and dispatch them to the appropriate handler based on the `method` field.

**Rationale:** JSON-RPC 2.0 is the mandatory protocol binding for A2A communication. All A2A methods are dispatched via this single endpoint.

**Acceptance Criteria:**
1. The server SHALL accept `Content-Type: application/json` POST requests at `/`.
2. The server SHALL validate that the request body conforms to JSON-RPC 2.0 structure (`jsonrpc`, `method`, `id`, `params`).
3. Malformed JSON SHALL return JSON-RPC error -32700 (Parse error).
4. Invalid JSON-RPC structure SHALL return JSON-RPC error -32600 (Invalid Request).
5. Unknown method names SHALL return JSON-RPC error -32601 (Method not found).
6. The server SHALL dispatch the following A2A methods: `message/send`, `message/stream`, `tasks/get`, `tasks/cancel`, `tasks/resubscribe`, `tasks/pushNotificationConfig/set`, `tasks/pushNotificationConfig/get`, `tasks/pushNotificationConfig/delete`.
7. Request body size SHALL be limited to 10 MB; requests exceeding this limit SHALL return HTTP 413.

---

#### FR-SRV-005: Server graceful shutdown preserves in-flight tasks

| Field | Value |
|-------|-------|
| **ID** | FR-SRV-005 |
| **Priority** | P0 |
| **PRD Trace** | FR-001 |

**Description:** Upon receiving SIGINT or SIGTERM, the server SHALL initiate graceful shutdown, allowing in-flight tasks to complete within a configurable grace period before forcefully terminating.

**Rationale:** Abrupt termination causes data loss and orphaned tasks. Graceful shutdown ensures reliability in production deployments.

**Acceptance Criteria:**
1. The server SHALL stop accepting new connections within 1 second of receiving SIGINT or SIGTERM.
2. In-flight tasks SHALL be allowed to complete for up to the configured grace period (default: 30 seconds).
3. Tasks not completed within the grace period SHALL be transitioned to `failed` state with message "Server shutdown".
4. Active SSE streams SHALL be terminated with a final `TaskStatusUpdateEvent` indicating server shutdown.
5. The grace period SHALL be configurable via `serve(registry, shutdown_timeout=60)`.

---

### 3.2 FR-AGC: Agent Card Module Requirements

---

#### FR-AGC-001: Generate Agent Card from Registry metadata

| Field | Value |
|-------|-------|
| **ID** | FR-AGC-001 |
| **Priority** | P0 |
| **PRD Trace** | FR-002 |

**Description:** The system SHALL automatically construct a valid A2A Agent Card from apcore Registry metadata, mapping Registry configuration fields to Agent Card fields according to the defined schema mapping.

**Rationale:** The Agent Card is the standard A2A discovery mechanism. Automatic generation from existing metadata eliminates manual authoring effort and ensures consistency.

**Acceptance Criteria:**
1. The generated Agent Card SHALL conform to A2A v0.3.0 Agent Card JSON Schema.
2. `AgentCard.name` SHALL be populated from Registry config `project.name`; fallback to `"apcore-agent"` if not configured.
3. `AgentCard.description` SHALL be populated from Registry config `project.description`; fallback to auto-generated description listing module count (e.g., "apcore agent with 10 skills").
4. `AgentCard.version` SHALL be populated from Registry config `project.version`; fallback to `"0.0.0"`.
5. `AgentCard.url` SHALL be populated from the server's bound address (scheme + host + port).
6. `AgentCard.skills[]` SHALL contain one Skill per registered module (per FR-SKL requirements).
7. `AgentCard.defaultInputModes` SHALL include `"application/json"`.
8. `AgentCard.defaultOutputModes` SHALL include `"application/json"`.

---

#### FR-AGC-002: Compute Agent Card capabilities from module annotations

| Field | Value |
|-------|-------|
| **ID** | FR-AGC-002 |
| **Priority** | P0 |
| **PRD Trace** | FR-002 |

**Description:** The system SHALL compute the Agent Card `capabilities` object by inspecting the annotations and configuration of all registered modules.

**Rationale:** Capabilities inform A2A clients about supported interaction modes. Automatic computation from module metadata ensures accuracy without manual configuration.

**Acceptance Criteria:**
1. `capabilities.streaming` SHALL be `true` if at least one registered module supports streaming (determined by Executor capability).
2. `capabilities.pushNotifications` SHALL be `true` if push notifications are enabled via `serve(registry, push_notifications=True)`.
3. `capabilities.stateTransitionHistory` SHALL be `true` if the configured task store supports history recording.
4. If no modules support streaming AND push notifications are disabled AND history is not supported, `capabilities` SHALL still be present with all fields set to `false`.

---

#### FR-AGC-003: Serve Agent Card at well-known endpoint

| Field | Value |
|-------|-------|
| **ID** | FR-AGC-003 |
| **Priority** | P0 |
| **PRD Trace** | FR-002 |

**Description:** The server SHALL serve the Agent Card as a JSON document at the standard A2A well-known endpoint.

**Rationale:** `/.well-known/agent.json` is the A2A standard discovery endpoint. All A2A clients expect to find the Agent Card at this URL.

**Acceptance Criteria:**
1. `GET /.well-known/agent.json` SHALL return HTTP 200 with `Content-Type: application/json`.
2. The response body SHALL be a valid JSON document conforming to the A2A v0.3.0 Agent Card schema.
3. The response SHALL include `Cache-Control: max-age=300` header (5-minute cache).
4. The endpoint SHALL NOT require authentication.
5. The response time SHALL be less than 10ms (p99) under normal load (pre-computed card served from memory).

---

#### FR-AGC-004: Serve extended Agent Card at authenticated endpoint

| Field | Value |
|-------|-------|
| **ID** | FR-AGC-004 |
| **Priority** | P1 |
| **PRD Trace** | FR-014 |

**Description:** The system SHALL serve an extended Agent Card at an authenticated endpoint that includes additional Skills not visible on the public Agent Card.

**Rationale:** Modules with ACL restrictions or `requires_approval` annotations should only be discoverable by authenticated agents to prevent information leakage about protected capabilities.

**Acceptance Criteria:**
1. `GET /agent/authenticatedExtendedCard` SHALL return an extended Agent Card when valid authentication is provided.
2. The extended card SHALL include all public Skills plus Skills for modules marked with `requires_approval` annotation or ACL restrictions.
3. The endpoint SHALL return HTTP 401 without valid credentials when authentication is configured.
4. The extended card SHALL declare all security schemes the agent supports.
5. `agentCard/get` JSON-RPC method SHALL return the appropriate card based on authentication status of the caller.
6. When authentication is not configured, the extended endpoint SHALL return HTTP 404.

---

#### FR-AGC-005: Regenerate Agent Card on module changes

| Field | Value |
|-------|-------|
| **ID** | FR-AGC-005 |
| **Priority** | P2 |
| **PRD Trace** | FR-019 |

**Description:** The system SHALL regenerate the Agent Card when modules are added to or removed from the Registry at runtime.

**Rationale:** Dynamic registration requires the Agent Card to reflect the current state of available skills without server restart.

**Acceptance Criteria:**
1. When a module is added via `registry.register()`, the Agent Card SHALL reflect the new skill within 1 second.
2. When a module is removed via `registry.unregister()`, the Agent Card SHALL exclude the removed skill within 1 second.
3. Agent Card regeneration SHALL be thread-safe and SHALL NOT block concurrent request handling.
4. The Agent Card version or a change identifier SHALL be updated on each regeneration.

---

### 3.3 FR-SKL: Skill Mapping Module Requirements

---

#### FR-SKL-001: Convert apcore module to A2A Skill object

| Field | Value |
|-------|-------|
| **ID** | FR-SKL-001 |
| **Priority** | P0 |
| **PRD Trace** | FR-003 |

**Description:** The system SHALL convert each apcore `ModuleDescriptor` into an A2A Skill object, mapping module metadata fields to Skill fields according to the defined schema mapping.

**Rationale:** Skills are the atomic capability units in A2A Agent Cards. Each apcore module represents exactly one skill, and the mapping must preserve all discoverable metadata.

**Acceptance Criteria:**
1. Each registered module SHALL produce exactly one Skill in the Agent Card.
2. `Skill.id` SHALL equal the apcore `module_id`.
3. `Skill.name` SHALL be derived from `module_id` by replacing dots and underscores with spaces and applying title case (e.g., `image.resize` becomes `Image Resize`).
4. `Skill.description` SHALL equal the apcore module `description` field verbatim.
5. `Skill.tags[]` SHALL equal the apcore module `tags[]` array.
6. Modules with missing or empty `description` (empty string or `None`) SHALL be excluded from Skills, and a warning SHALL be logged identifying the excluded module by `module_id`.

---

#### FR-SKL-002: Map module examples to Skill examples

| Field | Value |
|-------|-------|
| **ID** | FR-SKL-002 |
| **Priority** | P0 |
| **PRD Trace** | FR-003 |

**Description:** The system SHALL map apcore module examples to A2A Skill examples, converting example titles and inputs to the A2A example format.

**Rationale:** Examples help A2A clients understand how to invoke a skill. Preserving apcore examples in A2A format maximizes discoverability.

**Acceptance Criteria:**
1. `Skill.examples[]` SHALL be generated from `module.examples[]`.
2. Each example's `name` SHALL equal `module_example.title`.
3. Each example's `input` SHALL contain the module example input serialized as a TextPart (JSON string representation).
4. Modules with zero examples SHALL produce a Skill with an empty `examples[]` array.
5. Modules with more than 10 examples SHALL include only the first 10 in the Skill.

---

#### FR-SKL-003: Declare input and output modes on Skills

| Field | Value |
|-------|-------|
| **ID** | FR-SKL-003 |
| **Priority** | P0 |
| **PRD Trace** | FR-003 |

**Description:** The system SHALL declare `inputModes` and `outputModes` on each Skill based on the module's schema types.

**Rationale:** Input/output modes inform A2A clients which content types the skill accepts and produces, enabling correct message construction.

**Acceptance Criteria:**
1. Skills with `input_schema` defined SHALL declare `inputModes: ["application/json"]`.
2. Skills with `output_schema` defined SHALL declare `outputModes: ["application/json"]`.
3. Skills with text-type schemas (root type `string` or single string property) SHALL additionally declare `text/plain` in the respective modes.
4. Skills with no `input_schema` SHALL declare `inputModes: ["text/plain"]`.
5. Skills with no `output_schema` SHALL declare `outputModes: ["text/plain"]`.

---

#### FR-SKL-004: Include module annotations as Skill extensions

| Field | Value |
|-------|-------|
| **ID** | FR-SKL-004 |
| **Priority** | P0 |
| **PRD Trace** | FR-003 |

**Description:** The system SHALL include apcore module annotations as extension metadata on A2A Skills, using the A2A extensions mechanism.

**Rationale:** A2A has no native annotation fields for readonly, destructive, etc. Using the extensions mechanism preserves this information for apcore-aware clients without breaking non-apcore clients.

**Acceptance Criteria:**
1. When `ModuleAnnotations` is present, the Skill SHALL include `extensions.apcore.annotations` with keys: `readonly`, `destructive`, `idempotent`, `requires_approval`, `open_world`.
2. Each annotation value SHALL be a boolean matching the apcore annotation value.
3. When `ModuleAnnotations` is `None`, the `extensions.apcore.annotations` key SHALL be omitted.
4. Non-apcore A2A clients SHALL be able to parse the Agent Card without error (extensions are ignored by clients that do not recognize them).

---

### 3.4 FR-MSG: Message Handling Module Requirements

---

#### FR-MSG-001: Handle message/send for synchronous task execution

| Field | Value |
|-------|-------|
| **ID** | FR-MSG-001 |
| **Priority** | P0 |
| **PRD Trace** | FR-004 |

**Description:** The system SHALL implement the `message/send` JSON-RPC method that accepts a Message, routes it to the appropriate apcore module, executes it through the Executor pipeline, and returns the completed Task with Artifacts.

**Rationale:** Synchronous message/send is the primary interaction mode for simple request-response operations. It must route through the full Executor pipeline to preserve security and validation guarantees.

**Acceptance Criteria:**
1. The system SHALL accept JSON-RPC 2.0 requests with method `message/send`.
2. The system SHALL extract the target module from `params.metadata.skillId`.
3. The system SHALL create a new Task with status `submitted` and unique `taskId` (UUID v4).
4. The system SHALL transition the Task to `working` before invoking the Executor.
5. On successful execution, the system SHALL transition the Task to `completed` with module output as Artifact.
6. On execution error, the system SHALL transition the Task to `failed` with error message in `Task.status.message`.
7. The system SHALL return the Task object in the JSON-RPC response result field.
8. If no `skillId` is provided in metadata, the system SHALL return JSON-RPC error -32602 (Invalid params) with message "Missing required parameter: metadata.skillId".
9. If the specified `skillId` does not match any registered module, the system SHALL return JSON-RPC error -32601 (Method not found) with message identifying the unknown skill ID.
10. The response SHALL be returned within `execution_timeout + 5ms` overhead ceiling (see NFR-PERF-002).

---

#### FR-MSG-002: Handle message/stream for SSE streaming execution

| Field | Value |
|-------|-------|
| **ID** | FR-MSG-002 |
| **Priority** | P0 |
| **PRD Trace** | FR-009 |

**Description:** The system SHALL implement the `message/stream` JSON-RPC method that returns an SSE stream for real-time task status updates and incremental artifact delivery.

**Rationale:** Streaming enables real-time progress monitoring for long-running operations and incremental result delivery, which is critical for interactive multi-agent workflows.

**Acceptance Criteria:**
1. `message/stream` SHALL return an HTTP response with `Content-Type: text/event-stream`.
2. A `TaskStatusUpdateEvent` SHALL be emitted on each state transition (submitted, working, completed, failed, canceled).
3. A `TaskArtifactUpdateEvent` SHALL be emitted when the module produces incremental output via `Executor.stream()`.
4. Events SHALL conform to A2A v0.3.0 SSE event schema.
5. For streaming-capable modules, intermediate output chunks from `Executor.stream()` SHALL produce `TaskArtifactUpdateEvent` with `append: true`.
6. For non-streaming modules, the stream SHALL emit status updates only (submitted, working, completed/failed) with the final result as an Artifact in the terminal event.
7. The final event SHALL always be a `TaskStatusUpdateEvent` with a terminal state (completed, failed, or canceled).
8. SSE events SHALL include an `id` field containing a monotonically increasing integer for resumability.
9. The first `TaskStatusUpdateEvent` (submitted) SHALL be emitted within 50ms of request receipt.

---

#### FR-MSG-003: Parse message Parts into module input

| Field | Value |
|-------|-------|
| **ID** | FR-MSG-003 |
| **Priority** | P0 |
| **PRD Trace** | FR-004 |

**Description:** The system SHALL parse A2A message Parts into apcore module input parameters, supporting both structured JSON and plain text content types.

**Rationale:** A2A messages carry content as Parts (TextPart, DataPart, FilePart). The bridge must correctly extract and transform this content into the format expected by apcore module input schemas.

**Acceptance Criteria:**
1. `DataPart` with media type `application/json` SHALL be deserialized to a Python dict and passed as module input.
2. `TextPart` content SHALL be passed as string input when the module's `input_schema` root type is `string`.
3. `TextPart` containing valid JSON SHALL be auto-parsed to a dict when the module's `input_schema` root type is `object`.
4. `TextPart` containing invalid JSON when the module expects an object SHALL return JSON-RPC error -32602 with message "Invalid JSON in TextPart".
5. Messages with zero Parts SHALL return JSON-RPC error -32602 with message "Message must contain at least one Part".
6. Messages with multiple Parts SHALL use the first Part matching the skill's declared `inputModes`; remaining Parts SHALL be stored in the context for multi-turn access.
7. `FilePart` SHALL be passed as a dict with keys `uri`, `name`, and `mimeType` to modules that accept file input.

---

#### FR-MSG-004: Convert module output to Artifact Parts

| Field | Value |
|-------|-------|
| **ID** | FR-MSG-004 |
| **Priority** | P0 |
| **PRD Trace** | FR-004 |

**Description:** The system SHALL convert apcore module execution output into A2A Artifact Parts.

**Rationale:** Module outputs are Python dicts or strings. These must be converted to the A2A Part format for inclusion in Task Artifacts.

**Acceptance Criteria:**
1. Dict output SHALL be converted to a `DataPart` with media type `application/json` and the dict serialized as the data field.
2. String output SHALL be converted to a `TextPart`.
3. Output containing a `bytes` value SHALL be converted to a `FilePart` with appropriate media type detection.
4. `None` output SHALL produce an empty Artifact (zero Parts).
5. Each Task SHALL contain exactly one Artifact upon completion (containing one or more Parts from the module output).

---

#### FR-MSG-005: Client disconnection handling during streaming

| Field | Value |
|-------|-------|
| **ID** | FR-MSG-005 |
| **Priority** | P0 |
| **PRD Trace** | FR-009 |

**Description:** The system SHALL detect client disconnection during SSE streaming and take configurable action.

**Rationale:** Clients may disconnect mid-stream due to network issues or intentional cancellation. The server must handle this gracefully to avoid resource leaks.

**Acceptance Criteria:**
1. The server SHALL detect client disconnection within 5 seconds via TCP connection monitoring.
2. When `cancel_on_disconnect=True` (default), client disconnection SHALL trigger task cancellation via CancelToken.
3. When `cancel_on_disconnect=False`, the task SHALL continue executing in the background; the client may reconnect via `tasks/resubscribe`.
4. Resource cleanup (stream buffers, event queues) SHALL occur within 1 second of disconnection detection.

---

#### FR-MSG-006: Support tasks/resubscribe for SSE reconnection

| Field | Value |
|-------|-------|
| **ID** | FR-MSG-006 |
| **Priority** | P0 |
| **PRD Trace** | FR-009 |

**Description:** The system SHALL implement the `tasks/resubscribe` JSON-RPC method allowing clients to reconnect to an active task's SSE event stream.

**Rationale:** Network interruptions should not force clients to lose visibility into task progress. Resubscription enables resilient streaming connections.

**Acceptance Criteria:**
1. `tasks/resubscribe` with `{"id": "<taskId>"}` SHALL return a new SSE stream for the specified task.
2. The stream SHALL emit a `TaskStatusUpdateEvent` with the current task state as the first event.
3. Subsequent events SHALL continue from the current point (no replay of past events).
4. Resubscribing to a task in a terminal state (completed, failed, canceled) SHALL return a single `TaskStatusUpdateEvent` with the terminal state and then close the stream.
5. Resubscribing to a non-existent task SHALL return JSON-RPC error -32001 (TaskNotFound).

---

### 3.5 FR-TSK: Task Management Module Requirements

---

#### FR-TSK-001: Implement A2A task state machine

| Field | Value |
|-------|-------|
| **ID** | FR-TSK-001 |
| **Priority** | P0 |
| **PRD Trace** | FR-005 |

**Description:** The system SHALL implement the A2A task state machine governing all task state transitions with exactly the valid transitions defined by the A2A v0.3.0 specification.

**Rationale:** The task state machine is the stateful core of the A2A server. Invalid transitions would violate protocol compliance and cause interoperability failures.

**Acceptance Criteria:**
1. The system SHALL support exactly six task states: `submitted`, `working`, `completed`, `failed`, `canceled`, `input_required`.
2. Valid transitions SHALL be enforced:
   - `submitted` --> `working`, `canceled`, `failed`
   - `working` --> `completed`, `failed`, `canceled`, `input_required`
   - `input_required` --> `working`, `canceled`, `failed`
3. Terminal states (`completed`, `failed`, `canceled`) SHALL NOT allow any outbound transitions.
4. Attempted invalid transitions SHALL be logged at ERROR level and SHALL NOT be exposed to the client.
5. Attempted invalid transitions SHALL raise an internal `InvalidStateTransitionError` that is caught and converted to JSON-RPC error -32603.

---

#### FR-TSK-002: Task object structure

| Field | Value |
|-------|-------|
| **ID** | FR-TSK-002 |
| **Priority** | P0 |
| **PRD Trace** | FR-005 |

**Description:** Each Task object SHALL contain the fields required by the A2A v0.3.0 specification.

**Rationale:** Compliance with the A2A Task schema ensures interoperability with all A2A clients and orchestrators.

**Acceptance Criteria:**
1. Each Task SHALL have a unique `id` field (UUID v4, generated at creation time).
2. Each Task SHALL have a `contextId` field (UUID v4, provided by caller or auto-generated).
3. Each Task SHALL have a `status` object containing: `state` (string), `message` (optional string), `timestamp` (ISO 8601 string).
4. Each Task SHALL have an `artifacts` array (empty until completed; contains Artifact objects).
5. Each Task SHALL have a `history` array recording previous status objects when state transition history is enabled.
6. `status.timestamp` SHALL be updated on every state transition using UTC time in ISO 8601 format.

---

#### FR-TSK-003: Thread-safe concurrent task access

| Field | Value |
|-------|-------|
| **ID** | FR-TSK-003 |
| **Priority** | P0 |
| **PRD Trace** | FR-005 |

**Description:** All task state transitions and task store operations SHALL be thread-safe, supporting concurrent access from multiple request handlers.

**Rationale:** An async HTTP server processes multiple requests concurrently. Without thread-safe task access, race conditions could corrupt task state.

**Acceptance Criteria:**
1. State transitions on a single task SHALL be atomic (no intermediate state visible to other readers).
2. Concurrent state transitions on the same task SHALL be serialized (only one transition succeeds; others receive the post-transition state).
3. Concurrent access to different tasks SHALL NOT block each other.
4. The system SHALL support at least 100 simultaneous active tasks without deadlock or data corruption.

---

#### FR-TSK-004: Implement tasks/get for task state retrieval

| Field | Value |
|-------|-------|
| **ID** | FR-TSK-004 |
| **Priority** | P0 |
| **PRD Trace** | FR-008 |

**Description:** The system SHALL implement the `tasks/get` JSON-RPC method for retrieving the current state of a task by ID.

**Rationale:** Clients must be able to poll task state for non-streaming interactions and verify completion status.

**Acceptance Criteria:**
1. `tasks/get` with `{"id": "<taskId>"}` SHALL return the full Task object including status, artifacts, and history.
2. `tasks/get` for a non-existent task ID SHALL return JSON-RPC error -32001 with message "Task not found: <taskId>".
3. `tasks/get` SHALL include the `history` array when state transition history is enabled.
4. The response SHALL reflect the most recent task state (no stale reads).

---

#### FR-TSK-005: Implement tasks/cancel for task cancellation

| Field | Value |
|-------|-------|
| **ID** | FR-TSK-005 |
| **Priority** | P0 |
| **PRD Trace** | FR-008 |

**Description:** The system SHALL implement the `tasks/cancel` JSON-RPC method for canceling in-flight tasks.

**Rationale:** Long-running tasks must be cancellable to support workflow management and resource cleanup.

**Acceptance Criteria:**
1. `tasks/cancel` with `{"id": "<taskId>"}` SHALL transition a `submitted` or `working` or `input_required` task to `canceled` state.
2. Cancellation SHALL trigger the apcore CancelToken to abort the underlying Executor call.
3. `tasks/cancel` on a task in terminal state (`completed`, `failed`, `canceled`) SHALL return JSON-RPC error -32002 with message "Task is not cancelable: current state is <state>".
4. `tasks/cancel` on a non-existent task SHALL return JSON-RPC error -32001 (TaskNotFound).
5. The canceled task's `status.message` SHALL be set to "Canceled by client".

---

#### FR-TSK-006: Implement tasks/list with filtering and pagination

| Field | Value |
|-------|-------|
| **ID** | FR-TSK-006 |
| **Priority** | P0 |
| **PRD Trace** | FR-008 |

**Description:** The system SHALL implement task listing with optional contextId filtering and cursor-based pagination.

**Rationale:** Clients managing multi-turn conversations need to retrieve all tasks within a context. Pagination prevents excessive response sizes.

**Acceptance Criteria:**
1. `tasks/list` without parameters SHALL return all tasks ordered by creation time (newest first).
2. `tasks/list` with `{"contextId": "<id>"}` SHALL return only tasks belonging to the specified context.
3. Pagination SHALL be supported via `cursor` (opaque string) and `limit` (integer) parameters.
4. Default `limit` SHALL be 50; maximum `limit` SHALL be 200.
5. Requests with `limit` exceeding 200 SHALL be clamped to 200 (not rejected).
6. The response SHALL include a `nextCursor` field when more results are available.
7. The response SHALL include a `tasks` array containing Task objects.

---

### 3.6 FR-EXE: Execution Routing Module Requirements

---

#### FR-EXE-001: Route A2A messages through apcore Executor pipeline

| Field | Value |
|-------|-------|
| **ID** | FR-EXE-001 |
| **Priority** | P0 |
| **PRD Trace** | FR-006 |

**Description:** The system SHALL route incoming A2A messages to the appropriate apcore module through the full Executor pipeline, preserving ACL enforcement, input validation, middleware execution, and timeout handling.

**Rationale:** The Executor pipeline is the security and quality guarantee layer. Bypassing it would expose modules to unauthorized access, invalid input, and uncontrolled execution.

**Acceptance Criteria:**
1. The system SHALL invoke `Executor.call_async(module_id, inputs, context)` for synchronous execution.
2. The system SHALL invoke `Executor.stream(module_id, inputs, context)` for streaming execution.
3. `Executor.validate(module_id, inputs)` SHALL be called before execution when the module has an `input_schema`.
4. Validation failures SHALL return JSON-RPC error -32602 (Invalid params) with field-level details in the `data` field.
5. The full Executor pipeline (ACL, validation, middleware, timeout) SHALL be invoked for every task execution.
6. Execution timeout SHALL be configurable (default: 300 seconds).

---

#### FR-EXE-002: Map ApprovalPendingError to input_required state

| Field | Value |
|-------|-------|
| **ID** | FR-EXE-002 |
| **Priority** | P0 |
| **PRD Trace** | FR-006 |

**Description:** When a module raises `ApprovalPendingError`, the system SHALL transition the task to `input_required` state instead of treating it as an error.

**Rationale:** `ApprovalPendingError` indicates a human-in-the-loop approval is needed. The A2A `input_required` state is semantically equivalent and enables the client to provide additional input.

**Acceptance Criteria:**
1. `ApprovalPendingError` SHALL cause the task to transition to `input_required` state.
2. The task's `status.message` SHALL describe what input is needed (e.g., "Approval required for module <module_id>").
3. The `input_required` state SHALL NOT be represented as a JSON-RPC error.
4. A follow-up `message/send` or `message/stream` with the same `contextId` and `taskId` SHALL resume the task from `input_required` to `working`.

---

#### FR-EXE-003: Populate Identity context from authenticated requests

| Field | Value |
|-------|-------|
| **ID** | FR-EXE-003 |
| **Priority** | P1 |
| **PRD Trace** | FR-006, FR-013 |

**Description:** When authentication is configured, the system SHALL extract identity information from the authenticated request and populate the apcore Identity context variable before invoking the Executor.

**Rationale:** The Executor's ACL system uses the Identity context to enforce access control. Without identity bridging, authenticated A2A requests would be treated as anonymous.

**Acceptance Criteria:**
1. JWT claims (`sub`, `email`, `name`, `roles`, custom claims) SHALL be extracted from validated tokens.
2. Extracted identity SHALL be set as apcore's Identity context variable via `ContextVar` bridge.
3. Modules SHALL be able to access the authenticated identity via standard apcore `context.identity` API.
4. When authentication is not configured, Identity context SHALL remain unset (anonymous access).

---

### 3.7 FR-ERR: Error Mapping Module Requirements

---

#### FR-ERR-001: Map ModuleNotFoundError to JSON-RPC -32601

| Field | Value |
|-------|-------|
| **ID** | FR-ERR-001 |
| **Priority** | P0 |
| **PRD Trace** | FR-007 |

**Description:** The system SHALL map apcore `ModuleNotFoundError` to JSON-RPC error code -32601 (Method not found).

**Rationale:** A missing module is semantically equivalent to a missing JSON-RPC method. The module ID should be included in the error message for debugging.

**Acceptance Criteria:**
1. `ModuleNotFoundError` SHALL produce JSON-RPC error with code -32601.
2. The error message SHALL include the requested module ID (e.g., "Skill not found: <module_id>").
3. The error `data` field SHALL include `{"type": "ModuleNotFoundError"}`.

---

#### FR-ERR-002: Map SchemaValidationError to JSON-RPC -32602

| Field | Value |
|-------|-------|
| **ID** | FR-ERR-002 |
| **Priority** | P0 |
| **PRD Trace** | FR-007 |

**Description:** The system SHALL map apcore `SchemaValidationError` to JSON-RPC error code -32602 (Invalid params) with field-level validation details.

**Rationale:** Validation errors must be actionable. Field-level details enable clients to correct their input without guessing.

**Acceptance Criteria:**
1. `SchemaValidationError` SHALL produce JSON-RPC error with code -32602.
2. The error `data` field SHALL include an `errors` array with objects containing `field`, `code`, and `message` for each validation failure.
3. The error `data` field SHALL include `{"type": "SchemaValidationError"}`.

---

#### FR-ERR-003: Map ACLDeniedError to JSON-RPC -32001 with detail suppression

| Field | Value |
|-------|-------|
| **ID** | FR-ERR-003 |
| **Priority** | P0 |
| **PRD Trace** | FR-007 |

**Description:** The system SHALL map apcore `ACLDeniedError` to JSON-RPC error code -32001 (TaskNotFound) with all security-sensitive details suppressed.

**Rationale:** ACL errors must not leak information about the existence of protected modules, caller identities, or ACL rules. Returning TaskNotFound prevents information disclosure.

**Acceptance Criteria:**
1. `ACLDeniedError` SHALL produce JSON-RPC error with code -32001.
2. The error message SHALL be a generic "Task not found" (not "Access denied").
3. The error SHALL NOT include caller_id, target_id, or ACL rule details in any field.
4. The error `data` field SHALL include `{"type": "TaskNotFoundError"}` (masking the true error type).
5. The actual ACL denial SHALL be logged at WARNING level with full details for server-side debugging.

---

#### FR-ERR-004: Map ModuleExecuteError to JSON-RPC -32603

| Field | Value |
|-------|-------|
| **ID** | FR-ERR-004 |
| **Priority** | P0 |
| **PRD Trace** | FR-007 |

**Description:** The system SHALL map apcore `ModuleExecuteError` to JSON-RPC error code -32603 (Internal error) with sanitized message.

**Rationale:** Internal execution errors must not expose stack traces, file paths, or internal configuration to external clients.

**Acceptance Criteria:**
1. `ModuleExecuteError` SHALL produce JSON-RPC error with code -32603.
2. The error message SHALL be sanitized: no stack traces, file paths, or internal configuration values.
3. The error `data` field SHALL include `{"type": "ModuleExecuteError"}`.

---

#### FR-ERR-005: Map ModuleTimeoutError to JSON-RPC -32603

| Field | Value |
|-------|-------|
| **ID** | FR-ERR-005 |
| **Priority** | P0 |
| **PRD Trace** | FR-007 |

**Description:** The system SHALL map apcore `ModuleTimeoutError` to JSON-RPC error code -32603 with a timeout-specific message.

**Rationale:** Timeout errors have distinct semantics from general execution errors. The message should indicate the timeout condition without exposing internal timeout configuration.

**Acceptance Criteria:**
1. `ModuleTimeoutError` SHALL produce JSON-RPC error with code -32603.
2. The error message SHALL be "Execution timed out".
3. The error `data` field SHALL include `{"type": "ModuleTimeoutError"}`.

---

#### FR-ERR-006: Map InvalidInputError to JSON-RPC -32602

| Field | Value |
|-------|-------|
| **ID** | FR-ERR-006 |
| **Priority** | P0 |
| **PRD Trace** | FR-007 |

**Description:** The system SHALL map apcore `InvalidInputError` to JSON-RPC error code -32602.

**Rationale:** Invalid input is semantically equivalent to invalid params in JSON-RPC.

**Acceptance Criteria:**
1. `InvalidInputError` SHALL produce JSON-RPC error with code -32602.
2. The error message SHALL include the input error description.
3. The error `data` field SHALL include `{"type": "InvalidInputError"}`.

---

#### FR-ERR-007: Map safety limit errors to JSON-RPC -32603

| Field | Value |
|-------|-------|
| **ID** | FR-ERR-007 |
| **Priority** | P0 |
| **PRD Trace** | FR-007 |

**Description:** The system SHALL map apcore safety limit errors (`CallDepthExceededError`, `CircularCallError`, `CallFrequencyExceededError`) to JSON-RPC error code -32603 with a generic safety message.

**Rationale:** Safety limit errors indicate internal execution constraints. The specific constraint details should not be exposed to external clients.

**Acceptance Criteria:**
1. `CallDepthExceededError` SHALL produce JSON-RPC error -32603 with message "Safety limit exceeded".
2. `CircularCallError` SHALL produce JSON-RPC error -32603 with message "Safety limit exceeded".
3. `CallFrequencyExceededError` SHALL produce JSON-RPC error -32603 with message "Safety limit exceeded".
4. The error `data.type` SHALL identify the specific error type for programmatic handling.

---

#### FR-ERR-008: Map unknown exceptions to generic JSON-RPC -32603

| Field | Value |
|-------|-------|
| **ID** | FR-ERR-008 |
| **Priority** | P0 |
| **PRD Trace** | FR-007 |

**Description:** The system SHALL map any unrecognized exception to JSON-RPC error code -32603 with a generic message.

**Rationale:** Unknown exceptions must never leak internal details. A generic error message ensures security while still indicating a server-side failure.

**Acceptance Criteria:**
1. Any exception not matched by FR-ERR-001 through FR-ERR-007 SHALL produce JSON-RPC error -32603.
2. The error message SHALL be "Internal error".
3. The error SHALL NOT include stack traces, file paths, or internal variable values.
4. The error `data` field SHALL include `{"type": "InternalError"}`.
5. The actual exception SHALL be logged at ERROR level with full stack trace for server-side debugging.

---

### 3.8 FR-CLI: Client Module Requirements

---

#### FR-CLI-001: Construct A2AClient from remote agent URL

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-001 |
| **Priority** | P0 |
| **PRD Trace** | FR-010 |

**Description:** The system SHALL provide an `A2AClient` class that constructs a client pointing to a remote A2A agent's base URL.

**Rationale:** apcore agents must be able to discover and invoke remote A2A agents, enabling participation in multi-agent workflows as both server and client.

**Acceptance Criteria:**
1. `A2AClient(url)` SHALL construct a client pointing to the specified base URL.
2. The URL SHALL be validated as a well-formed HTTP or HTTPS URL; invalid URLs SHALL raise `ValueError`.
3. `A2AClient(url, auth="Bearer <token>")` SHALL set the authentication header for all requests.
4. `A2AClient(url, timeout=60)` SHALL set the HTTP request timeout (default: 30 seconds).
5. The client SHALL use `httpx.AsyncClient` for HTTP requests.
6. The client SHALL be importable independently: `from apcore_a2a.client import A2AClient` SHALL NOT import server-side dependencies (starlette, uvicorn).

---

#### FR-CLI-002: Fetch and cache remote Agent Card

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-002 |
| **Priority** | P0 |
| **PRD Trace** | FR-010 |

**Description:** The `A2AClient` SHALL fetch and cache the Agent Card from the remote agent's well-known endpoint.

**Rationale:** Agent Card discovery is the first step in any A2A interaction. Caching prevents redundant HTTP requests for repeated interactions.

**Acceptance Criteria:**
1. `client.agent_card` property SHALL fetch the Agent Card from `<url>/.well-known/agent.json` on first access.
2. The Agent Card SHALL be cached with a configurable TTL (default: 5 minutes).
3. After TTL expiration, the next access SHALL trigger a fresh fetch.
4. HTTP errors during Agent Card fetch SHALL raise `A2ADiscoveryError` with the status code and URL.
5. Invalid JSON in the Agent Card response SHALL raise `A2ADiscoveryError` with a descriptive message.

---

#### FR-CLI-003: Send synchronous message to remote agent

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-003 |
| **Priority** | P0 |
| **PRD Trace** | FR-010 |

**Description:** The `A2AClient` SHALL provide a `send_message()` method for sending synchronous `message/send` requests to remote agents.

**Rationale:** Synchronous invocation is the most common interaction pattern and must be supported as a first-class client operation.

**Acceptance Criteria:**
1. `await client.send_message(message)` SHALL send a `message/send` JSON-RPC request to the remote agent.
2. The method SHALL return the Task object from the response.
3. JSON-RPC error responses SHALL raise typed exceptions: `TaskNotFoundError` for -32001, `TaskNotCancelableError` for -32002, `A2AServerError` for -32603.
4. HTTP-level errors (connection refused, timeout) SHALL raise `A2AConnectionError`.

---

#### FR-CLI-004: Stream messages from remote agent

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-004 |
| **Priority** | P0 |
| **PRD Trace** | FR-010 |

**Description:** The `A2AClient` SHALL provide a `stream_message()` method for sending `message/stream` requests and receiving SSE events as an async iterator.

**Rationale:** Streaming enables real-time progress monitoring for long-running remote operations.

**Acceptance Criteria:**
1. `async for event in client.stream_message(message)` SHALL send `message/stream` and yield SSE events.
2. Each yielded event SHALL be a parsed `TaskStatusUpdateEvent` or `TaskArtifactUpdateEvent` object.
3. The iterator SHALL terminate when the stream closes (final terminal event received).
4. Connection errors during streaming SHALL raise `A2AConnectionError`.

---

#### FR-CLI-005: Get and cancel remote tasks

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-005 |
| **Priority** | P0 |
| **PRD Trace** | FR-010 |

**Description:** The `A2AClient` SHALL provide methods for retrieving and canceling tasks on remote agents.

**Rationale:** Task management operations are essential for workflow orchestration, retry logic, and resource cleanup.

**Acceptance Criteria:**
1. `await client.get_task(task_id)` SHALL send `tasks/get` and return the Task object.
2. `await client.cancel_task(task_id)` SHALL send `tasks/cancel` and return the updated Task.
3. `await client.list_tasks(context_id=None)` SHALL send `tasks/list` with optional filter and return the task list.
4. All methods SHALL raise typed exceptions for A2A error codes.

---

### 3.9 FR-CTX: Context Module Requirements

---

#### FR-CTX-001: Generate and manage contextId for conversations

| Field | Value |
|-------|-------|
| **ID** | FR-CTX-001 |
| **Priority** | P0 |
| **PRD Trace** | FR-011 |

**Description:** The system SHALL support multi-turn conversations by managing `contextId` identifiers that group related messages.

**Rationale:** Multi-turn conversations are essential for interactive workflows, input elicitation, and iterative refinement. The contextId mechanism enables stateful interactions.

**Acceptance Criteria:**
1. `message/send` and `message/stream` SHALL accept an optional `contextId` parameter in `params`.
2. If `contextId` is provided, the message SHALL be added to the existing conversation context.
3. If `contextId` is not provided, a new `contextId` (UUID v4) SHALL be generated and assigned to the task.
4. The generated or provided `contextId` SHALL be included in the returned Task object.

---

#### FR-CTX-002: Store and retrieve conversation history

| Field | Value |
|-------|-------|
| **ID** | FR-CTX-002 |
| **Priority** | P0 |
| **PRD Trace** | FR-011 |

**Description:** The system SHALL store conversation history (previous messages) for each contextId and make it accessible to modules during execution.

**Rationale:** Multi-turn modules need access to previous messages to maintain conversational state and make contextually appropriate responses.

**Acceptance Criteria:**
1. All messages within a context SHALL be stored in chronological order.
2. Conversation history SHALL be accessible to the module via the Executor context during execution.
3. A new `contextId` SHALL start with an empty history.
4. Context history SHALL be bounded to a configurable maximum number of messages (default: 100).
5. When the maximum is reached, the oldest messages SHALL be evicted (FIFO).

---

#### FR-CTX-003: Resume input_required tasks via contextId

| Field | Value |
|-------|-------|
| **ID** | FR-CTX-003 |
| **Priority** | P0 |
| **PRD Trace** | FR-011 |

**Description:** The system SHALL allow resuming tasks in `input_required` state by sending a follow-up message with the same `contextId`.

**Rationale:** Input elicitation (e.g., approval requests) requires a mechanism for the client to provide additional input and resume the paused task.

**Acceptance Criteria:**
1. A `message/send` with the same `contextId` as an `input_required` task SHALL resume that task.
2. The resumed task SHALL transition from `input_required` to `working`.
3. The follow-up message content SHALL be available to the module as additional input.
4. `tasks/list` with `contextId` filter SHALL return all tasks in the conversation, including the resumed task.

---

#### FR-CTX-004: Context state persistence in task store

| Field | Value |
|-------|-------|
| **ID** | FR-CTX-004 |
| **Priority** | P0 |
| **PRD Trace** | FR-011 |

**Description:** Context state (conversation history, contextId mappings) SHALL be stored alongside tasks in the configured task store.

**Rationale:** Context state must be co-located with task state to ensure consistency and support the same persistence guarantees.

**Acceptance Criteria:**
1. Context state SHALL be stored in the configured `TaskStore` implementation.
2. In-memory task store SHALL store context state in memory (lost on restart).
3. Persistent task stores (custom implementations) SHALL persist context state across server restarts.
4. Context state SHALL be cleaned up when all tasks in a context reach terminal states and the context TTL expires.

---

### 3.10 FR-PSH: Push Notification Module Requirements

---

#### FR-PSH-001: Register webhook URL for push notifications

| Field | Value |
|-------|-------|
| **ID** | FR-PSH-001 |
| **Priority** | P1 |
| **PRD Trace** | FR-012 |

**Description:** The system SHALL implement `tasks/pushNotificationConfig/set` to register a webhook URL for task state change notifications.

**Rationale:** Push notifications enable async workflows where clients do not maintain persistent connections. Webhook URLs must be registered per-task.

**Acceptance Criteria:**
1. `tasks/pushNotificationConfig/set` with `{"taskId": "<id>", "url": "<webhook_url>"}` SHALL register the webhook.
2. Webhook URLs SHALL be validated as well-formed URLs.
3. In production mode, webhook URLs SHALL be required to use HTTPS; HTTP SHALL be rejected with error -32602.
4. In development mode (configurable), HTTP webhook URLs SHALL be allowed.
5. Push notification support SHALL be disabled by default; enabled via `serve(registry, push_notifications=True)`.
6. When push notifications are disabled, the method SHALL return JSON-RPC error -32601 (Method not found).

---

#### FR-PSH-002: Deliver webhook notifications on state transitions

| Field | Value |
|-------|-------|
| **ID** | FR-PSH-002 |
| **Priority** | P1 |
| **PRD Trace** | FR-012 |

**Description:** The system SHALL deliver HTTP POST requests to registered webhook URLs on each task state transition.

**Rationale:** Webhook delivery is the core mechanism of push notifications. Each state transition must trigger a delivery to keep the client informed.

**Acceptance Criteria:**
1. On each task state transition, an HTTP POST SHALL be sent to the registered webhook URL.
2. The POST body SHALL contain the event payload conforming to A2A v0.3.0 push notification schema.
3. The POST SHALL include `Content-Type: application/json` header.
4. Successful delivery SHALL be indicated by HTTP 2xx response from the webhook.
5. Webhook delivery SHALL be asynchronous and SHALL NOT block task execution.

---

#### FR-PSH-003: Retry failed webhook deliveries with exponential backoff

| Field | Value |
|-------|-------|
| **ID** | FR-PSH-003 |
| **Priority** | P1 |
| **PRD Trace** | FR-012 |

**Description:** Failed webhook deliveries SHALL be retried with exponential backoff.

**Rationale:** Transient network failures should not cause permanent notification loss. Exponential backoff prevents overwhelming the webhook endpoint.

**Acceptance Criteria:**
1. Failed deliveries (HTTP 4xx/5xx or connection error) SHALL be retried up to 3 times.
2. Retry delays SHALL follow exponential backoff: 1 second, 2 seconds, 4 seconds.
3. After 3 failed retries, the push notification configuration for that task SHALL be marked as `failed`.
4. Retry attempts SHALL be logged at WARNING level with the failure reason.

---

#### FR-PSH-004: Manage push notification configurations

| Field | Value |
|-------|-------|
| **ID** | FR-PSH-004 |
| **Priority** | P1 |
| **PRD Trace** | FR-012 |

**Description:** The system SHALL implement methods for querying and deleting push notification configurations.

**Rationale:** Clients must be able to inspect and remove webhook configurations for lifecycle management.

**Acceptance Criteria:**
1. `tasks/pushNotificationConfig/get` with `{"taskId": "<id>"}` SHALL return the current push notification configuration.
2. `tasks/pushNotificationConfig/delete` with `{"taskId": "<id>"}` SHALL remove the webhook configuration.
3. Getting or deleting a non-existent configuration SHALL return JSON-RPC error -32001 (TaskNotFound).

---

### 3.11 FR-AUT: Authentication Module Requirements

---

#### FR-AUT-001: JWT/Bearer token validation middleware

| Field | Value |
|-------|-------|
| **ID** | FR-AUT-001 |
| **Priority** | P1 |
| **PRD Trace** | FR-013 |

**Description:** The system SHALL implement authentication middleware that validates JWT/Bearer tokens on incoming A2A requests.

**Rationale:** Enterprise deployments require authenticated access to A2A agents. JWT is the most common token format for service-to-service authentication.

**Acceptance Criteria:**
1. `serve(registry, auth=JWTAuthenticator(key=..., issuer=..., audience=...))` SHALL enable JWT authentication.
2. Incoming requests with `Authorization: Bearer <token>` SHALL be validated for signature, expiration, issuer, and audience.
3. Requests without tokens SHALL receive HTTP 401 response with `WWW-Authenticate: Bearer` header.
4. Requests with invalid or expired tokens SHALL receive HTTP 401 response.
5. Token content SHALL NOT be leaked in error messages.

---

#### FR-AUT-002: Bridge JWT claims to apcore Identity

| Field | Value |
|-------|-------|
| **ID** | FR-AUT-002 |
| **Priority** | P1 |
| **PRD Trace** | FR-013 |

**Description:** The system SHALL extract JWT claims and bridge them to apcore's Identity context variable.

**Rationale:** apcore's ACL system uses Identity for access control. JWT claims must be mapped to Identity fields so ACL rules work transparently with A2A callers.

**Acceptance Criteria:**
1. JWT claims `sub`, `email`, `name`, `roles` SHALL be extracted from validated tokens.
2. Custom claims SHALL be accessible via the Identity object.
3. The Identity SHALL be set via apcore's `ContextVar` bridge before Executor invocation.
4. Modules SHALL access the identity via `context.identity` API.

---

#### FR-AUT-003: Declare security schemes in Agent Card

| Field | Value |
|-------|-------|
| **ID** | FR-AUT-003 |
| **Priority** | P1 |
| **PRD Trace** | FR-013 |

**Description:** When authentication is configured, the Agent Card SHALL declare the supported security schemes.

**Rationale:** A2A clients need to know what authentication schemes the agent accepts to provide correct credentials.

**Acceptance Criteria:**
1. When JWT auth is configured, `AgentCard.securitySchemes` SHALL include `{"type": "http", "scheme": "bearer"}`.
2. When API Key auth is configured, `AgentCard.securitySchemes` SHALL include `{"type": "apiKey", "in": "header", "name": "X-API-Key"}`.
3. When no auth is configured, `securitySchemes` SHALL be an empty array or omitted.

---

#### FR-AUT-004: Pluggable authentication protocol

| Field | Value |
|-------|-------|
| **ID** | FR-AUT-004 |
| **Priority** | P1 |
| **PRD Trace** | FR-013 |

**Description:** The authentication middleware SHALL be pluggable via an `Authenticator` protocol, allowing custom authentication backends.

**Rationale:** Different enterprises use different identity providers. A pluggable protocol enables integration with any authentication system.

**Acceptance Criteria:**
1. `Authenticator` SHALL be a `@runtime_checkable` Protocol with method `authenticate(headers: dict[str, str]) -> Identity | None`.
2. `JWTAuthenticator` SHALL be a concrete implementation of `Authenticator`.
3. Custom implementations SHALL be accepted via `serve(registry, auth=MyCustomAuth())`.
4. The system SHALL validate at startup that the provided auth object satisfies the `Authenticator` protocol.

---

### 3.12 FR-STR: Storage Module Requirements

---

#### FR-STR-001: Define TaskStore abstract interface

| Field | Value |
|-------|-------|
| **ID** | FR-STR-001 |
| **Priority** | P1 |
| **PRD Trace** | FR-015 |

**Description:** The system SHALL define a runtime-checkable `TaskStore` Protocol with methods for task CRUD operations and push notification configuration storage.

**Rationale:** A pluggable storage interface enables swapping the default in-memory store for persistent backends (Redis, PostgreSQL) in production without changing application code.

**Acceptance Criteria:**
1. `TaskStore` SHALL be a runtime-checkable Protocol with the following async methods:
   - `save(task: Task) -> None`
   - `get(task_id: str) -> Task | None`
   - `list(context_id: str | None, cursor: str | None, limit: int) -> TaskListResult`
   - `delete(task_id: str) -> bool`
2. `TaskStore` SHALL include methods for push notification config:
   - `save_push_config(task_id: str, config: PushNotificationConfig) -> None`
   - `get_push_config(task_id: str) -> PushNotificationConfig | None`
   - `delete_push_config(task_id: str) -> bool`
3. All methods SHALL be async (`async def`).

---

#### FR-STR-002: Implement InMemoryTaskStore

| Field | Value |
|-------|-------|
| **ID** | FR-STR-002 |
| **Priority** | P1 |
| **PRD Trace** | FR-015 |

**Description:** The system SHALL provide a default `InMemoryTaskStore` implementation that stores tasks in Python dictionaries.

**Rationale:** The in-memory store enables zero-configuration server startup for development and testing. Production deployments can swap in persistent stores.

**Acceptance Criteria:**
1. `InMemoryTaskStore` SHALL implement all `TaskStore` methods.
2. Tasks SHALL be stored in a Python dict keyed by `task_id`.
3. The store SHALL support TTL-based expiration (default: 1 hour, configurable).
4. The store SHALL have a configurable maximum capacity (default: 10,000 tasks).
5. When capacity is reached, the oldest expired tasks SHALL be evicted first; if no expired tasks exist, the oldest tasks SHALL be evicted.
6. All operations SHALL be thread-safe using asyncio locks.
7. Read operations (`get`, `list`) SHALL have O(1) or O(n) complexity where n is the number of matching tasks.

---

#### FR-STR-003: Custom TaskStore acceptance

| Field | Value |
|-------|-------|
| **ID** | FR-STR-003 |
| **Priority** | P1 |
| **PRD Trace** | FR-015 |

**Description:** The `serve()` function SHALL accept custom `TaskStore` implementations.

**Rationale:** Production deployments require persistent storage. The interface must be clearly defined so custom implementations can be validated.

**Acceptance Criteria:**
1. `serve(registry, task_store=MyCustomStore())` SHALL accept any object implementing the `TaskStore` interface.
2. The system SHALL validate at startup that the provided store implements all required methods.
3. If the store does not implement required methods, a `TypeError` SHALL be raised with a message listing missing methods.

---

### 3.13 FR-CMD: CLI Module Requirements

---

#### FR-CMD-001: CLI entry point for server launch

| Field | Value |
|-------|-------|
| **ID** | FR-CMD-001 |
| **Priority** | P1 |
| **PRD Trace** | FR-016 |

**Description:** The system SHALL provide an `apcore-a2a` CLI command and `python -m apcore_a2a` entry point for launching the A2A server from the command line.

**Rationale:** CLI access enables quick testing and deployment without writing Python code, lowering the barrier to entry.

**Acceptance Criteria:**
1. `apcore-a2a serve --extensions-dir ./extensions` SHALL discover modules and start the A2A server.
2. CLI SHALL support `--host` (default: `0.0.0.0`), `--port` (default: `8000`), `--name`, `--description`, `--version` options.
3. `--auth-type bearer --auth-issuer <url>` SHALL enable JWT authentication.
4. `--push-notifications` flag SHALL enable push notification support.
5. `--log-level` option SHALL accept `debug`, `info`, `warning`, `error` (default: `info`).
6. `apcore-a2a --version` SHALL print the package version and exit.
7. `apcore-a2a --help` SHALL print usage information and exit.
8. CLI SHALL be registered as a console entry point via `pyproject.toml` (`[project.scripts]`).

---

#### FR-CMD-002: CLI error handling and exit codes

| Field | Value |
|-------|-------|
| **ID** | FR-CMD-002 |
| **Priority** | P1 |
| **PRD Trace** | FR-016 |

**Description:** The CLI SHALL use standard exit codes and provide clear error messages for configuration problems.

**Rationale:** Standard exit codes enable integration with process managers and deployment scripts.

**Acceptance Criteria:**
1. Exit code 0 SHALL indicate clean shutdown.
2. Exit code 1 SHALL indicate configuration error (invalid arguments, missing directory).
3. Exit code 2 SHALL indicate runtime error (server crash, unrecoverable failure).
4. If `--extensions-dir` points to a non-existent directory, the CLI SHALL exit with code 1 and message "Extensions directory not found: <path>".
5. If no modules are discovered in the extensions directory, the CLI SHALL exit with code 1 and message "No modules discovered in <path>".

---

### 3.14 FR-EXP: Explorer Module Requirements

---

#### FR-EXP-001: Serve browser-based Explorer UI

| Field | Value |
|-------|-------|
| **ID** | FR-EXP-001 |
| **Priority** | P2 |
| **PRD Trace** | FR-017 |

**Description:** The system SHALL provide an optional, lightweight browser UI for exploring agent skills, composing test messages, and viewing task state.

**Rationale:** A built-in UI accelerates development and debugging by providing immediate visual feedback without requiring external tools.

**Acceptance Criteria:**
1. When `explorer=True` is passed to `serve()`, the Explorer UI SHALL be mounted at `/explorer` (configurable prefix).
2. `GET /explorer/` SHALL return an interactive HTML page displaying Agent Card metadata, Skills list, and test forms.
3. The Explorer SHALL display all Skills with descriptions, tags, examples, and input/output modes.
4. The Explorer SHALL provide a form to compose and send `message/send` requests with result display.
5. The Explorer SHALL support `message/stream` with live SSE event rendering.
6. The Explorer SHALL be a single self-contained HTML file with no external CDN dependencies.
7. The Explorer SHALL be disabled by default; it SHALL require explicit `explorer=True` opt-in.
8. Explorer GET endpoints SHALL be exempt from JWT authentication; POST endpoints SHALL enforce auth when configured.

---

### 3.15 FR-OPS: Operations Module Requirements

---

#### FR-OPS-001: Health check endpoint

| Field | Value |
|-------|-------|
| **ID** | FR-OPS-001 |
| **Priority** | P2 |
| **PRD Trace** | FR-018 |

**Description:** The system SHALL provide a health check endpoint for operational monitoring.

**Rationale:** Production deployments require health checks for load balancer integration and automated monitoring.

**Acceptance Criteria:**
1. `GET /health` SHALL return HTTP 200 with JSON body `{"status": "healthy", "module_count": N, "uptime_seconds": N}` when the server is operational.
2. The health endpoint SHALL check task store connectivity; if the store is unreachable, the response SHALL be `{"status": "unhealthy", "reason": "..."}` with HTTP 503.
3. The health endpoint SHALL NOT require authentication.
4. The health endpoint SHALL respond within 1 second.

---

#### FR-OPS-002: Metrics endpoint

| Field | Value |
|-------|-------|
| **ID** | FR-OPS-002 |
| **Priority** | P2 |
| **PRD Trace** | FR-018 |

**Description:** The system SHALL provide a metrics endpoint for operational insights.

**Rationale:** Metrics enable capacity planning, performance monitoring, and anomaly detection in production.

**Acceptance Criteria:**
1. `GET /metrics` SHALL return JSON with: `active_tasks` (int), `completed_tasks` (int), `failed_tasks` (int), `uptime_seconds` (float), `total_requests` (int).
2. Metrics SHALL be disabled by default; enabled via `serve(registry, metrics=True)`.
3. Metrics SHALL NOT include any PII or task content.
4. Metrics endpoint SHALL NOT require authentication.

---

#### FR-OPS-003: Dynamic skill registration at runtime

| Field | Value |
|-------|-------|
| **ID** | FR-OPS-003 |
| **Priority** | P2 |
| **PRD Trace** | FR-019 |

**Description:** The system SHALL support adding and removing modules from the Registry at runtime, with automatic Agent Card and routing table updates.

**Rationale:** Long-running server deployments benefit from hot-reload capabilities that avoid downtime when module configurations change.

**Acceptance Criteria:**
1. When a module is added to the Registry via `registry.register()`, it SHALL appear in the Agent Card Skills within 1 second.
2. When a module is removed via `registry.unregister()`, it SHALL be removed from the Agent Card Skills within 1 second.
3. In-flight tasks for removed modules SHALL complete normally.
4. New tasks targeting removed modules SHALL return JSON-RPC error -32601 (Method not found).
5. Hot-reload SHALL be triggered by Registry events (`registry.on("register", callback)`, `registry.on("unregister", callback)`).
6. Agent Card changes SHALL be thread-safe and SHALL NOT block concurrent request handling.
7. An optional file-system watcher mode SHALL re-scan the extensions directory on file changes.

---

## 4. Specific Requirements -- Non-Functional Requirements

### 4.1 NFR-PERF: Performance Requirements

---

#### NFR-PERF-001: Agent Card response time

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-001 |
| **Priority** | P0 |
| **PRD Trace** | NFR-001 |

**Description:** The system SHALL serve Agent Card responses with latency below 10ms at the 99th percentile.

**Rationale:** Agent Card is served from pre-computed memory. High latency would indicate an implementation defect.

**Acceptance Criteria:**
1. `GET /.well-known/agent.json` SHALL respond in less than 10ms (p99) under normal load.
2. Measured via load test: 1,000 concurrent requests against the Agent Card endpoint.
3. The Agent Card SHALL be pre-computed at startup and served from memory (not regenerated per request).

---

#### NFR-PERF-002: message/send latency overhead

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-002 |
| **Priority** | P0 |
| **PRD Trace** | NFR-001 |

**Description:** The system SHALL add less than 5ms of protocol overhead to message/send execution time, above the raw Executor.call_async() time.

**Rationale:** The adapter layer must be thin. Excessive overhead would make A2A invocation slower than direct Executor calls.

**Acceptance Criteria:**
1. `message/send` latency SHALL be less than `Executor.call_async()` time + 5ms.
2. Measured via benchmark: compare `message/send` round-trip versus direct Executor call on the same module.

---

#### NFR-PERF-003: SSE first event latency

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-003 |
| **Priority** | P0 |
| **PRD Trace** | NFR-001 |

**Description:** The system SHALL emit the first SSE event within 50ms of receiving a `message/stream` request.

**Rationale:** Fast first-event delivery provides immediate feedback to streaming clients and confirms the stream is active.

**Acceptance Criteria:**
1. Time from `message/stream` HTTP request receipt to first `TaskStatusUpdateEvent` (submitted) SHALL be less than 50ms.
2. Measured via integration test with timestamp comparison.

---

#### NFR-PERF-004: Task store read latency

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-004 |
| **Priority** | P0 |
| **PRD Trace** | NFR-001 |

**Description:** The in-memory task store SHALL serve read operations with latency below 1ms.

**Rationale:** Task reads occur frequently during polling and task management operations. Sub-millisecond reads are expected for in-memory stores.

**Acceptance Criteria:**
1. `tasks/get` SHALL respond in less than 1ms when using `InMemoryTaskStore` with up to 10,000 stored tasks.
2. Measured via benchmark with 10,000 tasks pre-loaded.

---

#### NFR-PERF-005: Concurrent task support

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-005 |
| **Priority** | P0 |
| **PRD Trace** | NFR-001 |

**Description:** The system SHALL support at least 100 simultaneous active tasks without performance degradation.

**Rationale:** Production multi-agent workflows may generate many concurrent tasks. The system must handle this without bottlenecks.

**Acceptance Criteria:**
1. The system SHALL process 100 parallel `message/send` requests without errors, deadlocks, or data corruption.
2. p99 response time with 100 concurrent tasks SHALL be within 2x of single-task response time (excluding module execution time).
3. Measured via load test with 100 parallel requests.

---

#### NFR-PERF-006: Memory overhead per task

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-006 |
| **Priority** | P0 |
| **PRD Trace** | NFR-001 |

**Description:** Each task SHALL consume less than 10KB of memory in the task store, excluding artifact content.

**Rationale:** With a default capacity of 10,000 tasks, memory overhead must be bounded to prevent out-of-memory conditions.

**Acceptance Criteria:**
1. Task metadata (id, contextId, status, history) SHALL consume less than 10KB per task.
2. Measured via memory profiling with 10,000 tasks under sustained load.

---

#### NFR-PERF-007: Agent Card generation time

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-007 |
| **Priority** | P0 |
| **PRD Trace** | NFR-001 |

**Description:** Agent Card generation from Registry metadata SHALL complete within 100ms for registries with up to 100 modules.

**Rationale:** Agent Card generation occurs at startup and on dynamic registration events. Sub-100ms generation ensures negligible impact on startup time and runtime responsiveness.

**Acceptance Criteria:**
1. Agent Card generation from a Registry with 100 modules SHALL complete in less than 100ms.
2. Measured via benchmark with 100 mock modules.

---

#### NFR-PERF-008: Server startup time

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-008 |
| **Priority** | P0 |
| **PRD Trace** | NFR-001 |

**Description:** The time from `serve()` call to the server being ready to accept the first request SHALL be less than 2 seconds.

**Rationale:** Fast startup enables rapid iteration during development and fast recovery from restarts in production.

**Acceptance Criteria:**
1. Time from `serve()` invocation to successful `GET /.well-known/agent.json` response SHALL be less than 2 seconds.
2. Measured via integration test with timestamp comparison.

---

### 4.2 NFR-SEC: Security Requirements

---

#### NFR-SEC-001: No information leakage in error responses

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-001 |
| **Priority** | P0 |
| **PRD Trace** | NFR-002 |

**Description:** Error responses SHALL NOT leak internal implementation details including stack traces, file paths, internal variable names, or configuration values.

**Rationale:** Information leakage enables reconnaissance attacks and violates security best practices.

**Acceptance Criteria:**
1. No error response SHALL contain Python stack traces.
2. No error response SHALL contain file system paths.
3. No error response SHALL contain internal variable names or configuration values.
4. ACL errors SHALL be masked as "Task not found" (per FR-ERR-003).
5. Verified via security scan of all error response paths.

---

#### NFR-SEC-002: Input sanitization

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-002 |
| **Priority** | P0 |
| **PRD Trace** | NFR-002 |

**Description:** All client-provided strings SHALL be sanitized before inclusion in logs or error messages.

**Rationale:** Prevents log injection attacks and ensures log integrity.

**Acceptance Criteria:**
1. Client-provided strings included in log messages SHALL be truncated to 1,000 characters.
2. Control characters (ASCII 0-31 except newline, tab) SHALL be stripped from logged client input.
3. Client-provided strings SHALL NOT be used in f-strings or format strings for error messages without escaping.

---

#### NFR-SEC-003: CORS configuration

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-003 |
| **Priority** | P0 |
| **PRD Trace** | NFR-002 |

**Description:** The server SHALL enforce CORS policy with configurable allowed origins.

**Rationale:** CORS prevents unauthorized cross-origin access to the A2A endpoint from browser-based clients.

**Acceptance Criteria:**
1. Default CORS policy SHALL be same-origin only (no `Access-Control-Allow-Origin` header).
2. `serve(registry, cors_origins=["https://example.com"])` SHALL set `Access-Control-Allow-Origin` to the specified origins.
3. `serve(registry, cors_origins=["*"])` SHALL allow all origins (development only; logged as WARNING).
4. CORS preflight `OPTIONS` requests SHALL be handled correctly with appropriate headers.

---

#### NFR-SEC-004: Webhook URL validation

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-004 |
| **Priority** | P1 |
| **PRD Trace** | NFR-002 |

**Description:** Webhook URLs registered for push notifications SHALL be validated to prevent SSRF attacks.

**Rationale:** Server-side request forgery (SSRF) via webhook URLs could enable attackers to probe internal networks.

**Acceptance Criteria:**
1. Webhook URLs SHALL be validated as well-formed HTTP/HTTPS URLs.
2. In production mode, only HTTPS URLs SHALL be accepted.
3. URLs pointing to loopback addresses (127.0.0.1, ::1, localhost) SHALL be rejected in production mode.
4. URLs pointing to private IP ranges (10.x, 172.16-31.x, 192.168.x) SHALL be rejected in production mode.

---

#### NFR-SEC-005: Dependency security

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-005 |
| **Priority** | P0 |
| **PRD Trace** | NFR-002 |

**Description:** All dependencies SHALL have no known critical CVEs at release time.

**Rationale:** Vulnerable dependencies are the most common attack vector for Python packages.

**Acceptance Criteria:**
1. `pip audit` SHALL report zero critical or high severity vulnerabilities before each release.
2. Dependency versions SHALL be pinned in `pyproject.toml` with compatible release specifiers (`>=`).

---

### 4.3 NFR-REL: Reliability Requirements

---

#### NFR-REL-001: Graceful shutdown

| Field | Value |
|-------|-------|
| **ID** | NFR-REL-001 |
| **Priority** | P0 |
| **PRD Trace** | NFR-003 |

**Description:** The server SHALL shut down gracefully, allowing in-flight tasks to complete within a configurable grace period.

**Rationale:** Abrupt termination causes data loss and orphaned tasks in production environments.

**Acceptance Criteria:**
1. In-flight tasks SHALL be allowed to complete for up to the grace period (default: 30 seconds).
2. New connections SHALL be rejected within 1 second of shutdown signal.
3. Tasks not completed within the grace period SHALL transition to `failed` with message "Server shutdown".
4. Server process SHALL exit cleanly after all tasks complete or the grace period expires.

---

#### NFR-REL-002: Error isolation

| Field | Value |
|-------|-------|
| **ID** | NFR-REL-002 |
| **Priority** | P0 |
| **PRD Trace** | NFR-003 |

**Description:** A failing module execution SHALL NOT crash the server or affect other in-flight tasks.

**Rationale:** Fault isolation is essential for multi-tenant server reliability. One bad module must not bring down the entire agent.

**Acceptance Criteria:**
1. An unhandled exception in module execution SHALL be caught, logged, and converted to a task `failed` state.
2. Other concurrent task executions SHALL continue unaffected.
3. The server process SHALL remain stable after any number of individual task failures.

---

#### NFR-REL-003: State consistency

| Field | Value |
|-------|-------|
| **ID** | NFR-REL-003 |
| **Priority** | P0 |
| **PRD Trace** | NFR-003 |

**Description:** Task state SHALL remain consistent under concurrent access with no invalid transitions.

**Rationale:** Inconsistent task state would violate the A2A protocol contract and confuse clients.

**Acceptance Criteria:**
1. The task state machine SHALL prevent all invalid transitions (as defined in FR-TSK-001).
2. Concurrent state transitions on the same task SHALL be serialized.
3. No task SHALL ever be observed in an inconsistent state (e.g., `completed` with empty artifacts when output was produced).

---

#### NFR-REL-004: Webhook resilience

| Field | Value |
|-------|-------|
| **ID** | NFR-REL-004 |
| **Priority** | P1 |
| **PRD Trace** | NFR-003 |

**Description:** Push notification webhook failures SHALL NOT block or delay task execution.

**Rationale:** Webhook delivery is best-effort. Task execution must proceed regardless of notification delivery status.

**Acceptance Criteria:**
1. Webhook delivery SHALL be asynchronous (fire-and-forget with retry).
2. Failed webhook delivery SHALL NOT delay task state transitions.
3. Failed webhook delivery SHALL NOT cause task failure.

---

### 4.4 NFR-CMP: Compatibility Requirements

---

#### NFR-CMP-001: A2A protocol compliance

| Field | Value |
|-------|-------|
| **ID** | NFR-CMP-001 |
| **Priority** | P0 |
| **PRD Trace** | NFR-004 |

**Description:** The system SHALL be fully compliant with A2A v0.3.0 specification.

**Rationale:** Protocol compliance is mandatory for interoperability with all A2A-compliant agents and clients.

**Acceptance Criteria:**
1. All A2A v0.3.0 JSON-RPC methods SHALL be implemented.
2. Agent Card SHALL validate against the A2A v0.3.0 Agent Card JSON Schema.
3. Task objects SHALL conform to A2A v0.3.0 Task schema.
4. SSE events SHALL conform to A2A v0.3.0 event schemas.
5. JSON-RPC 2.0 compliance SHALL be verified via protocol conformance tests.

---

#### NFR-CMP-002: apcore-python version compatibility

| Field | Value |
|-------|-------|
| **ID** | NFR-CMP-002 |
| **Priority** | P0 |
| **PRD Trace** | NFR-004 |

**Description:** The system SHALL be compatible with apcore-python >= 0.6.0.

**Rationale:** apcore-a2a must work with the current and future versions of the apcore SDK.

**Acceptance Criteria:**
1. The system SHALL import and use `apcore` package >= 0.6.0 without errors.
2. Duck-typing SHALL be used throughout (no hard type checks on apcore types) to maximize version tolerance.
3. Integration tests SHALL be run against at least two apcore-python versions (0.6.x and latest).

---

#### NFR-CMP-003: Python version support

| Field | Value |
|-------|-------|
| **ID** | NFR-CMP-003 |
| **Priority** | P0 |
| **PRD Trace** | NFR-004 |

**Description:** The system SHALL support Python >= 3.11.

**Rationale:** Aligns with apcore-python minimum version and enables modern async features.

**Acceptance Criteria:**
1. All tests SHALL pass on Python 3.11, 3.12, and 3.13.
2. No Python 3.11+ only syntax SHALL be used unless guarded by version checks.
3. CI/CD SHALL test against all supported Python versions.

---

#### NFR-CMP-004: ASGI server compatibility

| Field | Value |
|-------|-------|
| **ID** | NFR-CMP-004 |
| **Priority** | P0 |
| **PRD Trace** | NFR-004 |

**Description:** The ASGI application returned by `async_serve()` SHALL be compatible with any ASGI-compliant server.

**Rationale:** Production deployments may use different ASGI servers (uvicorn, hypercorn, daphne).

**Acceptance Criteria:**
1. The ASGI app SHALL work with uvicorn.
2. The ASGI app SHALL work with at least one additional ASGI server (hypercorn or daphne).
3. The ASGI app SHALL conform to the ASGI 3.0 specification.

---

#### NFR-CMP-005: Interoperability with non-apcore A2A agents

| Field | Value |
|-------|-------|
| **ID** | NFR-CMP-005 |
| **Priority** | P0 |
| **PRD Trace** | NFR-004 |

**Description:** The system SHALL interoperate with non-apcore A2A agents and clients.

**Rationale:** A2A is an open standard. apcore-a2a agents must work with agents built on any framework.

**Acceptance Criteria:**
1. The A2A Client SHALL successfully discover and invoke at least one non-apcore A2A agent.
2. The A2A Server SHALL successfully serve requests from at least one non-apcore A2A client.
3. Interoperability SHALL be verified via integration tests against A2A reference implementations.

---

### 4.5 NFR-MNT: Maintainability Requirements

---

#### NFR-MNT-001: Test coverage

| Field | Value |
|-------|-------|
| **ID** | NFR-MNT-001 |
| **Priority** | P0 |
| **PRD Trace** | NFR-005 |

**Description:** The system SHALL maintain high test coverage on core logic.

**Rationale:** Comprehensive tests prevent regression and enable confident refactoring.

**Acceptance Criteria:**
1. Line coverage SHALL be >= 90% on `src/apcore_a2a/` (excluding tests).
2. Branch coverage SHALL be >= 85% on `src/apcore_a2a/`.
3. Target test count SHALL be ~400-500 tests (comparable to apcore-mcp-python's ~450).

---

#### NFR-MNT-002: Type annotations

| Field | Value |
|-------|-------|
| **ID** | NFR-MNT-002 |
| **Priority** | P0 |
| **PRD Trace** | NFR-005 |

**Description:** All public APIs SHALL be fully type-annotated and pass mypy strict mode.

**Rationale:** Type annotations improve code quality, enable IDE support, and catch type errors at development time.

**Acceptance Criteria:**
1. 100% of public API functions and methods SHALL have type annotations for all parameters and return values.
2. `mypy --strict` SHALL pass with zero errors on the public API surface.
3. Internal implementation SHALL have at least 80% type annotation coverage.

---

#### NFR-MNT-003: Code documentation

| Field | Value |
|-------|-------|
| **ID** | NFR-MNT-003 |
| **Priority** | P0 |
| **PRD Trace** | NFR-005 |

**Description:** All public APIs SHALL have docstrings with usage examples.

**Rationale:** Documentation enables adoption and reduces support burden.

**Acceptance Criteria:**
1. Every public class, function, and method SHALL have a docstring.
2. Each docstring SHALL include at least one usage example.
3. Docstrings SHALL follow Google style or NumPy style consistently throughout the codebase.

---

#### NFR-MNT-004: Package size constraint

| Field | Value |
|-------|-------|
| **ID** | NFR-MNT-004 |
| **Priority** | P0 |
| **PRD Trace** | NFR-005 |

**Description:** Core logic SHALL not exceed 1,500 lines of code.

**Rationale:** The thin adapter design principle ensures complexity remains in apcore-python and a2a-sdk, not in the adapter.

**Acceptance Criteria:**
1. `cloc src/apcore_a2a/ --exclude-dir=tests` SHALL report <= 1,500 lines of Python code.
2. Explorer UI HTML assets are excluded from this count.

---

#### NFR-MNT-005: Architectural conformance

| Field | Value |
|-------|-------|
| **ID** | NFR-MNT-005 |
| **Priority** | P0 |
| **PRD Trace** | NFR-005 |

**Description:** The system SHALL follow the layered architecture pattern established by apcore-mcp-python.

**Rationale:** Consistent architecture across adapter projects reduces cognitive load for contributors and enables shared patterns.

**Acceptance Criteria:**
1. Source code SHALL be organized into directories: `adapters/`, `server/`, `auth/`, `store/`, `client/`.
2. Each layer SHALL have clear import boundaries (no circular imports between layers).
3. The adapter layer SHALL have no dependency on the server layer.
4. The client layer SHALL have no dependency on the server, auth, or store layers.

---

### 4.6 NFR-SCA: Scalability Requirements

---

#### NFR-SCA-001: Connection limits

| Field | Value |
|-------|-------|
| **ID** | NFR-SCA-001 |
| **Priority** | P0 |
| **PRD Trace** | NFR-001 |

**Description:** The system SHALL support at least 100 concurrent HTTP connections without degradation.

**Rationale:** Multi-agent workflows may generate many concurrent connections from different orchestrators and agents.

**Acceptance Criteria:**
1. The server SHALL accept and process at least 100 concurrent HTTP connections.
2. Connection handling SHALL not exhibit head-of-line blocking (each connection processed independently).
3. Measured via load test with 100 concurrent connections.

---

#### NFR-SCA-002: Task throughput

| Field | Value |
|-------|-------|
| **ID** | NFR-SCA-002 |
| **Priority** | P0 |
| **PRD Trace** | NFR-001 |

**Description:** The system SHALL support at least 100 task creations per second under sustained load.

**Rationale:** High-throughput multi-agent scenarios require the server to handle many tasks without becoming a bottleneck.

**Acceptance Criteria:**
1. The system SHALL process at least 100 `message/send` requests per second (excluding module execution time).
2. Task store operations SHALL not become a bottleneck at this throughput level.
3. Measured via load test with no-op modules (isolating adapter overhead).

---

#### NFR-SCA-003: SSE stream limits

| Field | Value |
|-------|-------|
| **ID** | NFR-SCA-003 |
| **Priority** | P0 |
| **PRD Trace** | NFR-001 |

**Description:** The system SHALL support at least 50 concurrent SSE streams without degradation.

**Rationale:** Streaming is resource-intensive (persistent connections, event buffering). A defined limit ensures predictable behavior.

**Acceptance Criteria:**
1. The system SHALL maintain at least 50 concurrent SSE streams with event delivery within 100ms.
2. Each SSE stream SHALL consume less than 1KB of server memory for buffering (excluding artifact content).
3. Measured via load test with 50 concurrent `message/stream` requests.

---

## 5. Use Cases

### UC-001: Synchronous Message Send and Task Completion

| Field | Value |
|-------|-------|
| **UC ID** | UC-001 |
| **Name** | Synchronous message/send with task completion |
| **Actors** | Agent Orchestrator (Omar), apcore-a2a Server |
| **PRD Trace** | US-003, FR-004, FR-005, FR-006, FR-007 |

**Preconditions:**
1. apcore-a2a server is running with at least one registered module (skill).
2. Orchestrator knows the server's base URL.
3. Orchestrator has a valid `skillId` for the target module.

**Main Flow:**
1. Orchestrator sends HTTP POST to `/` with JSON-RPC 2.0 body: `{"jsonrpc":"2.0","id":"1","method":"message/send","params":{"message":{"role":"user","parts":[{"type":"text","text":"resize image to 800x600"}]},"metadata":{"skillId":"image.resize"}}}`.
2. Server validates JSON-RPC structure.
3. Server extracts `skillId` from `params.metadata.skillId`.
4. Server creates Task with `status.state = "submitted"`, unique `taskId`, and auto-generated `contextId`.
5. Server transitions Task to `status.state = "working"`.
6. Server parses message Parts into module input (TextPart content as string input).
7. Server invokes `Executor.call_async("image.resize", inputs)`.
8. Executor runs full pipeline (ACL, validation, middleware, execution).
9. Module returns output dict.
10. Server converts output dict to DataPart artifact.
11. Server transitions Task to `status.state = "completed"` with Artifact.
12. Server returns JSON-RPC response with the completed Task object.

**Alternate Flows:**

*A1: Module not found*
- At step 3, if `skillId` does not match any registered module, server returns JSON-RPC error -32601 with message "Skill not found: image.resize".

*A2: Input validation failure*
- At step 8, if `Executor.validate()` fails, server returns JSON-RPC error -32602 with field-level validation details.

*A3: No skillId provided*
- At step 3, if `metadata.skillId` is missing, server returns JSON-RPC error -32602 with message "Missing required parameter: metadata.skillId".

**Exception Flows:**

*E1: Module execution error*
- At step 9, if the module raises `ModuleExecuteError`, server transitions Task to `failed` state with sanitized error message and returns the failed Task in the response.

*E2: Execution timeout*
- At step 9, if execution exceeds the timeout (default: 300s), server transitions Task to `failed` with message "Execution timed out" and returns the failed Task.

*E3: ACL denied*
- At step 8, if ACL denies access, server returns JSON-RPC error -32001 with generic "Task not found" message (no information leakage).

**Postconditions:**
1. Task is stored in the task store with terminal state (`completed` or `failed`).
2. Task is retrievable via `tasks/get` with the returned `taskId`.

---

### UC-002: Streaming Execution with SSE Events

| Field | Value |
|-------|-------|
| **UC ID** | UC-002 |
| **Name** | Streaming execution via message/stream with SSE |
| **Actors** | Agent Orchestrator (Omar), apcore-a2a Server |
| **PRD Trace** | US-004, FR-009, FR-005, FR-006 |

**Preconditions:**
1. apcore-a2a server is running with at least one streaming-capable module.
2. Agent Card `capabilities.streaming` is `true`.
3. Orchestrator has a valid `skillId` for a streaming module.

**Main Flow:**
1. Orchestrator sends HTTP POST to `/` with `message/stream` method.
2. Server validates request and creates Task with `status.state = "submitted"`.
3. Server returns HTTP response with `Content-Type: text/event-stream`.
4. Server emits `TaskStatusUpdateEvent` with `state: "submitted"`, `id: 1`.
5. Server transitions Task to `working` and emits `TaskStatusUpdateEvent` with `state: "working"`, `id: 2`.
6. Server invokes `Executor.stream(module_id, inputs)`.
7. For each chunk yielded by `Executor.stream()`:
   - Server emits `TaskArtifactUpdateEvent` with the chunk content and `append: true`, incrementing `id`.
8. When `Executor.stream()` completes:
   - Server transitions Task to `completed`.
   - Server emits final `TaskStatusUpdateEvent` with `state: "completed"`, containing the final Artifact.
9. Server closes the SSE stream.

**Alternate Flows:**

*A1: Non-streaming module*
- At step 6, if the module does not support streaming, server invokes `Executor.call_async()` instead.
- Only `TaskStatusUpdateEvent` events are emitted (submitted, working, completed/failed).
- No `TaskArtifactUpdateEvent` events are emitted during execution.

*A2: Client disconnection*
- At step 7, if the client disconnects and `cancel_on_disconnect=True`, server cancels the task via CancelToken and transitions to `canceled`.
- If `cancel_on_disconnect=False`, execution continues in background; client may reconnect via `tasks/resubscribe`.

**Exception Flows:**

*E1: Module error during streaming*
- At step 7, if the module raises an exception mid-stream, server transitions Task to `failed` and emits a final `TaskStatusUpdateEvent` with `state: "failed"` and the sanitized error message.

**Postconditions:**
1. Task is in terminal state (completed, failed, or canceled).
2. All emitted SSE events have sequential `id` values.
3. The final event is always a `TaskStatusUpdateEvent` with terminal state.

---

### UC-003: Multi-Turn Conversation with Input Elicitation

| Field | Value |
|-------|-------|
| **UC ID** | UC-003 |
| **Name** | Multi-turn conversation with input_required state |
| **Actors** | Agent Orchestrator (Omar), apcore-a2a Server |
| **PRD Trace** | US-006, FR-011, FR-005, FR-006 |

**Preconditions:**
1. apcore-a2a server is running with a module that has `requires_approval` annotation.
2. Orchestrator has a valid `skillId` for the approval-requiring module.

**Main Flow:**
1. Orchestrator sends `message/send` with `skillId` for an approval-requiring module (no explicit `contextId`).
2. Server creates Task, auto-generates `contextId`, transitions to `submitted`, then `working`.
3. Executor invokes the module; module raises `ApprovalPendingError`.
4. Server transitions Task to `input_required` with `status.message = "Approval required for module <module_id>"`.
5. Server returns Task with `status.state = "input_required"` and the generated `contextId`.
6. Orchestrator sends second `message/send` with the same `contextId` and approval confirmation in the message content.
7. Server matches the `contextId` to the `input_required` task.
8. Server transitions Task from `input_required` to `working`.
9. Module resumes execution with the approval input.
10. Module completes; server transitions Task to `completed` with Artifact.
11. Server returns the completed Task.

**Alternate Flows:**

*A1: Orchestrator cancels instead of approving*
- At step 6, orchestrator sends `tasks/cancel` instead of a follow-up message.
- Server transitions Task to `canceled`.

*A2: Context history access*
- At step 9, the module accesses `context.history` to read the original message and the approval message.

**Exception Flows:**

*E1: Invalid contextId in follow-up*
- At step 7, if the `contextId` does not match any existing context, server creates a new context and task (no error; treated as a new conversation).

*E2: Follow-up to non-input_required task*
- At step 7, if the matching task is not in `input_required` state, a new task is created within the same context.

**Postconditions:**
1. Task is in terminal state (`completed` or `canceled`).
2. Context contains two messages (original and follow-up).
3. Both messages are retrievable via `tasks/list` with `contextId` filter.

---

### UC-004: Client Discovery and Invocation of Remote Agent

| Field | Value |
|-------|-------|
| **UC ID** | UC-004 |
| **Name** | A2AClient discovers and invokes a remote A2A agent |
| **Actors** | Module Developer (Maya), Remote A2A Agent |
| **PRD Trace** | US-007, FR-010 |

**Preconditions:**
1. Remote A2A agent is running and accessible at a known URL.
2. `apcore_a2a.client` package is installed.

**Main Flow:**
1. Maya creates client: `client = A2AClient("https://remote-agent.example.com")`.
2. Maya fetches Agent Card: `card = await client.agent_card`.
3. Client sends GET to `https://remote-agent.example.com/.well-known/agent.json`.
4. Client receives and caches the Agent Card (TTL: 5 minutes).
5. Maya inspects skills: `print(card.skills)`.
6. Maya sends message: `task = await client.send_message(message)`.
7. Client sends `message/send` JSON-RPC request to the remote agent.
8. Remote agent processes the request and returns a completed Task.
9. Maya reads result: `print(task.status.state)` (prints "completed").

**Alternate Flows:**

*A1: Streaming invocation*
- At step 6, Maya uses `async for event in client.stream_message(message)` instead.
- Client sends `message/stream` and yields SSE events as they arrive.

*A2: Cached Agent Card*
- At step 3, if the Agent Card was fetched within the last 5 minutes, the cached version is returned without an HTTP request.

*A3: Task management*
- After step 9, Maya queries task: `task = await client.get_task(task.id)`.
- Or cancels: `await client.cancel_task(task.id)`.

**Exception Flows:**

*E1: Remote agent unreachable*
- At step 3 or 7, if the remote agent is unreachable, `A2AConnectionError` is raised with connection details.

*E2: Invalid Agent Card*
- At step 4, if the Agent Card JSON is malformed, `A2ADiscoveryError` is raised.

*E3: Remote execution error*
- At step 8, if the remote agent returns a JSON-RPC error, the client raises a typed exception (e.g., `TaskNotFoundError`, `A2AServerError`).

**Postconditions:**
1. Agent Card is cached for subsequent operations.
2. Task result is available to Maya for further processing.

---

### UC-005: Push Notification for Asynchronous Task Monitoring

| Field | Value |
|-------|-------|
| **UC ID** | UC-005 |
| **Name** | Push notification webhook for async task monitoring |
| **Actors** | Agent Orchestrator (Omar), apcore-a2a Server, Webhook Endpoint |
| **PRD Trace** | US-008, FR-012 |

**Preconditions:**
1. apcore-a2a server is running with `push_notifications=True`.
2. Agent Card `capabilities.pushNotifications` is `true`.
3. Orchestrator has a webhook endpoint URL accessible from the server.

**Main Flow:**
1. Orchestrator sends `message/send` to start a long-running task.
2. Server creates Task and returns it with `taskId`.
3. Orchestrator sends `tasks/pushNotificationConfig/set` with `{"taskId": "<id>", "url": "https://hooks.orchestrator.com/a2a"}`.
4. Server validates the webhook URL (HTTPS, not loopback).
5. Server stores the push notification configuration.
6. Task progresses: submitted -> working -> completed.
7. On each state transition, server sends HTTP POST to `https://hooks.orchestrator.com/a2a` with event payload.
8. Webhook endpoint acknowledges with HTTP 200.

**Alternate Flows:**

*A1: Webhook delivery failure and retry*
- At step 7, if the webhook endpoint returns HTTP 500, server retries after 1s.
- Second retry after 2s if still failing.
- Third retry after 4s if still failing.
- After 3 failures, the push notification config is marked as `failed`.

*A2: Orchestrator removes webhook*
- At any point, orchestrator sends `tasks/pushNotificationConfig/delete` to stop notifications.
- No further notifications are sent for that task.

*A3: Query webhook config*
- Orchestrator sends `tasks/pushNotificationConfig/get` to verify the current configuration.

**Exception Flows:**

*E1: Push notifications disabled*
- At step 3, if push notifications are not enabled, server returns JSON-RPC error -32601 (Method not found).

*E2: HTTP webhook URL in production*
- At step 4, if the URL uses HTTP (not HTTPS) in production mode, server returns JSON-RPC error -32602 with message "Webhook URL must use HTTPS in production mode".

**Postconditions:**
1. Orchestrator received webhook notifications for each state transition.
2. Push notification configuration is stored and retrievable.

---

### UC-006: Authenticated Agent Access with JWT

| Field | Value |
|-------|-------|
| **UC ID** | UC-006 |
| **Name** | JWT-authenticated agent access with ACL enforcement |
| **Actors** | Enterprise Architect (Elena), Agent Orchestrator (Omar), apcore-a2a Server |
| **PRD Trace** | US-009, FR-013, FR-014 |

**Preconditions:**
1. apcore-a2a server is running with `auth=JWTAuth(issuer="https://idp.corp.com", audience="apcore-agents")`.
2. Agent Card declares `securitySchemes: [{"type": "http", "scheme": "bearer"}]`.
3. Orchestrator has a valid JWT token from the configured issuer.

**Main Flow:**
1. Orchestrator fetches public Agent Card at `GET /.well-known/agent.json` (no auth required).
2. Orchestrator notes `securitySchemes` and obtains a Bearer token from the corporate IdP.
3. Orchestrator fetches extended Agent Card at `GET /agent/authenticatedExtendedCard` with `Authorization: Bearer <token>`.
4. Server validates JWT (signature, expiration, issuer, audience).
5. Server returns extended Agent Card with additional privileged skills.
6. Orchestrator sends `message/send` with `Authorization: Bearer <token>` targeting a privileged skill.
7. Server validates JWT and extracts claims (`sub`, `roles`).
8. Server bridges identity to apcore Identity context.
9. Executor invokes module with ACL check against the authenticated identity.
10. ACL permits; module executes and returns result.
11. Server returns completed Task.

**Alternate Flows:**

*A1: Expired token*
- At step 4 or 7, if the token is expired, server returns HTTP 401 with `WWW-Authenticate: Bearer error="invalid_token"`.
- Orchestrator refreshes token and retries.

*A2: ACL denied*
- At step 9, if ACL denies the identity, server returns JSON-RPC error -32001 with generic "Task not found" message (no security info leak).

**Exception Flows:**

*E1: Missing token*
- At step 6, if no `Authorization` header is present, server returns HTTP 401 with `WWW-Authenticate: Bearer`.

*E2: Invalid token format*
- At step 7, if the token is malformed (not a valid JWT), server returns HTTP 401. Token content is not leaked in the error response.

**Postconditions:**
1. Only authorized agents could invoke protected modules.
2. Identity flowed through to apcore ACL system.
3. Task history records the authenticated identity if history is enabled.

---

## 6. CRUD Matrix

The following matrix maps data entities to operations across the system's modules.

| Entity | Create | Read | Update | Delete |
|--------|:------:|:----:|:------:|:------:|
| **Task** | FR-MSG-001 (message/send creates task), FR-MSG-002 (message/stream creates task) | FR-TSK-004 (tasks/get), FR-TSK-006 (tasks/list) | FR-TSK-001 (state transitions), FR-TSK-005 (tasks/cancel) | FR-STR-002 (TTL expiration) |
| **Message** | FR-MSG-001 (incoming message), FR-MSG-003 (Part parsing) | FR-CTX-002 (context history read) | -- | FR-CTX-002 (FIFO eviction at max history) |
| **Artifact** | FR-MSG-004 (output conversion) | FR-TSK-004 (tasks/get returns artifacts) | FR-MSG-002 (append during streaming) | -- |
| **AgentCard** | FR-AGC-001 (generation at startup) | FR-AGC-003 (GET /.well-known/agent.json), FR-AGC-004 (extended card) | FR-AGC-005 (regeneration on module changes) | -- |
| **Skill** | FR-SKL-001 (module to skill conversion) | FR-AGC-003 (via Agent Card) | FR-OPS-003 (hot-reload updates) | FR-OPS-003 (removal on module unregister) |
| **PushNotificationConfig** | FR-PSH-001 (set webhook) | FR-PSH-004 (get config) | FR-PSH-003 (mark as failed after retries) | FR-PSH-004 (delete config) |
| **Context** | FR-CTX-001 (auto-generated contextId) | FR-CTX-002 (history retrieval) | FR-CTX-002 (append messages) | FR-CTX-004 (cleanup on TTL) |
| **Identity** | FR-AUT-002 (extracted from JWT) | FR-EXE-003 (read by Executor for ACL) | -- | -- (request-scoped) |

---

## 7. Data Requirements

### 7.1 Entity Definitions

#### 7.1.1 Task

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | string (UUID v4) | Required, unique, immutable | Unique task identifier generated at creation |
| `contextId` | string (UUID v4) | Required, immutable per task | Conversation context grouping identifier |
| `status` | TaskStatus object | Required | Current task status (see 7.1.2) |
| `artifacts` | array of Artifact | Required (may be empty) | Output artifacts produced by the task |
| `history` | array of TaskStatus | Optional | Previous status records (when enabled) |
| `metadata` | object | Optional | Request metadata including `skillId` |

#### 7.1.2 TaskStatus

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `state` | enum string | Required; one of: submitted, working, completed, failed, canceled, input_required | Current state |
| `message` | string | Optional | Human-readable status message |
| `timestamp` | string (ISO 8601) | Required | UTC timestamp of last transition |

#### 7.1.3 Artifact

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `name` | string | Optional | Artifact name |
| `parts` | array of Part | Required, at least one | Content parts |
| `index` | integer | Required | Artifact index (0-based) |
| `append` | boolean | Optional, default false | Whether this appends to a previous artifact |
| `lastChunk` | boolean | Optional | Whether this is the final chunk of a streaming artifact |

#### 7.1.4 Part (union type)

**TextPart:**

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `type` | string | Required, value: "text" | Part type discriminator |
| `text` | string | Required | Text content |

**DataPart:**

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `type` | string | Required, value: "data" | Part type discriminator |
| `data` | object | Required | Structured data content |
| `mediaType` | string | Required | MIME type (e.g., "application/json") |

**FilePart:**

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `type` | string | Required, value: "file" | Part type discriminator |
| `uri` | string (URI) | Required | File location URI |
| `name` | string | Optional | File name |
| `mimeType` | string | Optional | File MIME type |

#### 7.1.5 AgentCard

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `name` | string | Required | Agent display name |
| `description` | string | Required | Agent description |
| `version` | string | Required | Agent version (semver) |
| `url` | string (URL) | Required | Agent base URL |
| `skills` | array of Skill | Required, at least one | Agent capabilities |
| `capabilities` | Capabilities object | Required | Supported interaction modes |
| `defaultInputModes` | array of string | Required | Default input MIME types |
| `defaultOutputModes` | array of string | Required | Default output MIME types |
| `securitySchemes` | array of SecurityScheme | Optional | Supported auth schemes |

#### 7.1.6 Skill

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | string | Required, unique within card | Skill identifier (equals module_id) |
| `name` | string | Required | Human-readable skill name |
| `description` | string | Required | Skill description |
| `tags` | array of string | Optional | Categorization tags |
| `examples` | array of Example | Optional, max 10 | Usage examples |
| `inputModes` | array of string | Optional | Accepted input MIME types |
| `outputModes` | array of string | Optional | Produced output MIME types |
| `extensions` | object | Optional | Custom extension metadata |

#### 7.1.7 PushNotificationConfig

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `taskId` | string (UUID v4) | Required | Associated task ID |
| `url` | string (URL) | Required, HTTPS in production | Webhook delivery URL |
| `status` | enum string | Required; one of: active, failed | Configuration status |

#### 7.1.8 Context

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `contextId` | string (UUID v4) | Required, unique | Context identifier |
| `messages` | array of Message | Required | Chronological message history |
| `createdAt` | string (ISO 8601) | Required | Context creation timestamp |
| `updatedAt` | string (ISO 8601) | Required | Last message timestamp |

### 7.2 Entity Relationships

```
AgentCard 1 --- * Skill          (Agent Card contains many Skills)
Skill    1 --- 1 ModuleDescriptor (each Skill maps to one apcore module)
Task     * --- 1 Context          (many Tasks belong to one Context)
Task     1 --- * Artifact         (one Task produces many Artifacts)
Artifact 1 --- * Part             (one Artifact contains many Parts)
Task     1 --- 0..1 PushNotificationConfig (optional webhook per task)
Context  1 --- * Message          (one Context contains many Messages)
Message  1 --- * Part             (one Message contains many Parts)
```

### 7.3 Task State Machine

```
                           +-----------+
                           |           |
                    +----->| canceled  |
                    |      |           |
                    |      +-----------+
                    |
+----------+    +----------+    +-----------+
|          |--->|          |--->|           |
| submitted|    | working  |    | completed |
|          |    |          |    |           |
+----------+    +----------+    +-----------+
    |   |           |   |
    |   |           |   +-----> +-----------+
    |   |           |           |           |
    |   |           +---------->|  failed   |
    |   |           |           |           |
    |   +---------->+           +-----------+
    |               |
    +----------+    +-----> +----------------+
               |            |                |
               +----------->| input_required |
                            |                |
                            +-------+--------+
                                    |
                                    | (resume via follow-up message)
                                    |
                                    v
                               +----------+
                               | working  |
                               +----------+
```

**State Transition Table:**

| Current State | Event | Next State | Guard | Action |
|---------------|-------|------------|-------|--------|
| submitted | execution_start | working | -- | Invoke Executor |
| submitted | cancel_request | canceled | -- | Set status.message = "Canceled by client" |
| submitted | internal_error | failed | -- | Set status.message with sanitized error |
| working | execution_success | completed | Output is not None | Create Artifact from output |
| working | execution_error | failed | -- | Set status.message with sanitized error |
| working | cancel_request | canceled | CancelToken active | Trigger CancelToken.cancel() |
| working | approval_pending | input_required | Module raises ApprovalPendingError | Set status.message describing needed input |
| working | timeout | failed | Execution exceeds timeout | Set status.message = "Execution timed out" |
| input_required | resume_message | working | Follow-up message in same context | Provide new input to module |
| input_required | cancel_request | canceled | -- | Set status.message = "Canceled by client" |
| input_required | internal_error | failed | -- | Set status.message with sanitized error |

**Boundary Conditions:**
- Maximum tasks in store: 10,000 (configurable via `InMemoryTaskStore(max_capacity=N)`).
- Maximum message size: 10 MB per JSON-RPC request body.
- Maximum context history: 100 messages per context (configurable).
- Task TTL in in-memory store: 1 hour (configurable).
- Maximum concurrent active tasks: 100 (soft limit, governed by system resources).
- Maximum artifact size: bounded by available memory (no hard limit; documented as deployment consideration).

---

## 8. Interface Requirements

### 8.1 External Interfaces

#### 8.1.1 A2A JSON-RPC Endpoints

| Endpoint | Method | Direction | Description |
|----------|--------|-----------|-------------|
| `POST /` | `message/send` | Client -> Server | Synchronous message execution; returns completed Task |
| `POST /` | `message/stream` | Client -> Server | Streaming execution; returns SSE event stream |
| `POST /` | `tasks/get` | Client -> Server | Retrieve task by ID |
| `POST /` | `tasks/cancel` | Client -> Server | Cancel in-flight task |
| `POST /` | `tasks/resubscribe` | Client -> Server | Reconnect to active task SSE stream |
| `POST /` | `tasks/pushNotificationConfig/set` | Client -> Server | Register webhook URL |
| `POST /` | `tasks/pushNotificationConfig/get` | Client -> Server | Get webhook configuration |
| `POST /` | `tasks/pushNotificationConfig/delete` | Client -> Server | Remove webhook configuration |

All JSON-RPC endpoints accept `Content-Type: application/json` and return `Content-Type: application/json` (except `message/stream` and `tasks/resubscribe` which return `Content-Type: text/event-stream`).

#### 8.1.2 REST Endpoints

| Endpoint | Method | Auth Required | Description |
|----------|--------|:---:|-------------|
| `GET /.well-known/agent.json` | GET | No | Public Agent Card |
| `GET /agent/authenticatedExtendedCard` | GET | Yes | Extended Agent Card with privileged skills |
| `GET /health` | GET | No | Health check (when enabled) |
| `GET /metrics` | GET | No | Metrics (when enabled) |
| `GET /explorer/` | GET | No | Explorer UI (when enabled) |

#### 8.1.3 SSE Stream Events

| Event Type | Direction | Description |
|------------|-----------|-------------|
| `TaskStatusUpdateEvent` | Server -> Client | Emitted on each task state transition |
| `TaskArtifactUpdateEvent` | Server -> Client | Emitted when module produces incremental output |

SSE event format:
```
id: <sequential_integer>
data: {"type": "<event_type>", "taskId": "<id>", ...}

```

#### 8.1.4 Push Notification Webhook

| Direction | Method | Content-Type | Description |
|-----------|--------|:---:|-------------|
| Server -> Webhook | POST | application/json | Task state change notification |

Webhook payload conforms to A2A v0.3.0 push notification schema.

### 8.2 Software Interfaces

#### 8.2.1 apcore Registry API (consumed)

```python
registry.discover() -> int
registry.list(tags=None, prefix=None) -> list[str]
registry.get(module_id) -> module | None
registry.get_definition(module_id) -> ModuleDescriptor | None
registry.iter() -> Iterator[tuple[str, Any]]
registry.on("register", callback)
registry.on("unregister", callback)
```

#### 8.2.2 apcore Executor API (consumed)

```python
executor = Executor(registry, middlewares=None, acl=None, config=None)
await executor.call_async(module_id, inputs, context=None) -> dict
async for chunk in executor.stream(module_id, inputs, context=None): ...
executor.validate(module_id, inputs) -> ValidationResult
```

#### 8.2.3 apcore ModuleDescriptor API (consumed)

```python
descriptor.module_id: str
descriptor.description: str
descriptor.input_schema: dict[str, Any]
descriptor.output_schema: dict[str, Any]
descriptor.annotations: ModuleAnnotations | None
descriptor.name: str | None
descriptor.tags: list[str]
descriptor.version: str
descriptor.examples: list[ModuleExample]
```

#### 8.2.4 apcore Error Hierarchy (consumed)

```python
ModuleNotFoundError(module_id)
SchemaValidationError(message, errors)
ACLDeniedError(caller_id, target_id)
ModuleExecuteError(message)
ModuleTimeoutError(module_id, timeout_ms)
InvalidInputError(message)
CallDepthExceededError(depth, max_depth, call_chain)
CircularCallError(module_id, call_chain)
CallFrequencyExceededError(module_id, count, max_repeat, call_chain)
ApprovalPendingError(module_id)
```

#### 8.2.5 a2a-sdk API (consumed)

The system consumes A2A types, JSON-RPC handling, and SSE utilities from the `a2a-sdk` package. Specific API surface depends on SDK version; the adapter layer abstracts SDK usage behind internal interfaces.

### 8.3 Communication Interfaces

| Protocol | Direction | Purpose | Port |
|----------|-----------|---------|------|
| HTTP/HTTPS | Bidirectional | JSON-RPC requests, Agent Card, REST endpoints | Configurable (default: 8000) |
| SSE over HTTP/HTTPS | Server -> Client | Streaming event delivery | Same as HTTP |
| HTTP/HTTPS | Server -> Webhook | Push notification delivery | Webhook endpoint port |

**TLS:** TLS termination is expected to be handled by a reverse proxy (nginx, caddy, cloud load balancer). The apcore-a2a server itself listens on plain HTTP. This is documented as a deployment consideration.

---

## 9. Traceability Matrix

### 9.1 PRD Feature to SRS Requirement Mapping

| PRD Feature | PRD Title | Priority | SRS Requirements |
|-------------|-----------|----------|------------------|
| FR-001 | serve() -- One-Call A2A Server | P0 | FR-SRV-001, FR-SRV-002, FR-SRV-003, FR-SRV-004, FR-SRV-005 |
| FR-002 | Agent Card Auto-Generation | P0 | FR-AGC-001, FR-AGC-002, FR-AGC-003 |
| FR-003 | Skill Mapping | P0 | FR-SKL-001, FR-SKL-002, FR-SKL-003, FR-SKL-004 |
| FR-004 | message/send | P0 | FR-MSG-001, FR-MSG-003, FR-MSG-004 |
| FR-005 | Task Lifecycle (State Machine) | P0 | FR-TSK-001, FR-TSK-002, FR-TSK-003 |
| FR-006 | Execution Routing | P0 | FR-EXE-001, FR-EXE-002, FR-EXE-003 |
| FR-007 | Error Mapping | P0 | FR-ERR-001, FR-ERR-002, FR-ERR-003, FR-ERR-004, FR-ERR-005, FR-ERR-006, FR-ERR-007, FR-ERR-008 |
| FR-008 | tasks/get, tasks/list, tasks/cancel | P0 | FR-TSK-004, FR-TSK-005, FR-TSK-006 |
| FR-009 | SSE Streaming (message/stream) | P0 | FR-MSG-002, FR-MSG-005, FR-MSG-006 |
| FR-010 | A2A Client | P0 | FR-CLI-001, FR-CLI-002, FR-CLI-003, FR-CLI-004, FR-CLI-005 |
| FR-011 | Multi-Turn Conversations (contextId) | P0 | FR-CTX-001, FR-CTX-002, FR-CTX-003, FR-CTX-004 |
| FR-012 | Push Notifications | P1 | FR-PSH-001, FR-PSH-002, FR-PSH-003, FR-PSH-004 |
| FR-013 | JWT/Bearer Authentication | P1 | FR-AUT-001, FR-AUT-002, FR-AUT-003, FR-AUT-004 |
| FR-014 | Extended Agent Card | P1 | FR-AGC-004 |
| FR-015 | Task Storage Interface | P1 | FR-STR-001, FR-STR-002, FR-STR-003 |
| FR-016 | CLI Entry Point | P1 | FR-CMD-001, FR-CMD-002 |
| FR-017 | Agent Explorer UI | P2 | FR-EXP-001 |
| FR-018 | Health/Metrics Endpoints | P2 | FR-OPS-001, FR-OPS-002 |
| FR-019 | Dynamic Skill Registration | P2 | FR-AGC-005, FR-OPS-003 |

### 9.2 SRS Requirement to PRD Feature Reverse Mapping

| SRS Requirement | PRD Feature | Priority |
|-----------------|-------------|----------|
| FR-SRV-001 | FR-001 | P0 |
| FR-SRV-002 | FR-001 | P0 |
| FR-SRV-003 | FR-001 | P0 |
| FR-SRV-004 | FR-001 | P0 |
| FR-SRV-005 | FR-001 | P0 |
| FR-AGC-001 | FR-002 | P0 |
| FR-AGC-002 | FR-002 | P0 |
| FR-AGC-003 | FR-002 | P0 |
| FR-AGC-004 | FR-014 | P1 |
| FR-AGC-005 | FR-019 | P2 |
| FR-SKL-001 | FR-003 | P0 |
| FR-SKL-002 | FR-003 | P0 |
| FR-SKL-003 | FR-003 | P0 |
| FR-SKL-004 | FR-003 | P0 |
| FR-MSG-001 | FR-004 | P0 |
| FR-MSG-002 | FR-009 | P0 |
| FR-MSG-003 | FR-004 | P0 |
| FR-MSG-004 | FR-004 | P0 |
| FR-MSG-005 | FR-009 | P0 |
| FR-MSG-006 | FR-009 | P0 |
| FR-TSK-001 | FR-005 | P0 |
| FR-TSK-002 | FR-005 | P0 |
| FR-TSK-003 | FR-005 | P0 |
| FR-TSK-004 | FR-008 | P0 |
| FR-TSK-005 | FR-008 | P0 |
| FR-TSK-006 | FR-008 | P0 |
| FR-EXE-001 | FR-006 | P0 |
| FR-EXE-002 | FR-006 | P0 |
| FR-EXE-003 | FR-006, FR-013 | P1 |
| FR-ERR-001 | FR-007 | P0 |
| FR-ERR-002 | FR-007 | P0 |
| FR-ERR-003 | FR-007 | P0 |
| FR-ERR-004 | FR-007 | P0 |
| FR-ERR-005 | FR-007 | P0 |
| FR-ERR-006 | FR-007 | P0 |
| FR-ERR-007 | FR-007 | P0 |
| FR-ERR-008 | FR-007 | P0 |
| FR-CLI-001 | FR-010 | P0 |
| FR-CLI-002 | FR-010 | P0 |
| FR-CLI-003 | FR-010 | P0 |
| FR-CLI-004 | FR-010 | P0 |
| FR-CLI-005 | FR-010 | P0 |
| FR-CTX-001 | FR-011 | P0 |
| FR-CTX-002 | FR-011 | P0 |
| FR-CTX-003 | FR-011 | P0 |
| FR-CTX-004 | FR-011 | P0 |
| FR-PSH-001 | FR-012 | P1 |
| FR-PSH-002 | FR-012 | P1 |
| FR-PSH-003 | FR-012 | P1 |
| FR-PSH-004 | FR-012 | P1 |
| FR-AUT-001 | FR-013 | P1 |
| FR-AUT-002 | FR-013 | P1 |
| FR-AUT-003 | FR-013 | P1 |
| FR-AUT-004 | FR-013 | P1 |
| FR-STR-001 | FR-015 | P1 |
| FR-STR-002 | FR-015 | P1 |
| FR-STR-003 | FR-015 | P1 |
| FR-CMD-001 | FR-016 | P1 |
| FR-CMD-002 | FR-016 | P1 |
| FR-EXP-001 | FR-017 | P2 |
| FR-OPS-001 | FR-018 | P2 |
| FR-OPS-002 | FR-018 | P2 |
| FR-OPS-003 | FR-019 | P2 |

### 9.3 NFR Traceability

| SRS NFR | PRD NFR | Category |
|---------|---------|----------|
| NFR-PERF-001 through NFR-PERF-008 | NFR-001 | Performance |
| NFR-SEC-001 through NFR-SEC-005 | NFR-002 | Security |
| NFR-REL-001 through NFR-REL-004 | NFR-003 | Reliability |
| NFR-CMP-001 through NFR-CMP-005 | NFR-004 | Compatibility |
| NFR-MNT-001 through NFR-MNT-005 | NFR-005 | Maintainability |
| NFR-SCA-001 through NFR-SCA-003 | NFR-001, NFR-003 | Scalability |

---

## 10. Appendices

### Appendix A: Glossary

| Term | Definition |
|------|-----------|
| **A2A** | Agent-to-Agent protocol. Open standard by Google/Linux Foundation for inter-agent communication. Enables diverse AI agents to discover, communicate, and collaborate on tasks. |
| **Agent Card** | JSON document describing an A2A agent's identity, capabilities, skills, and security schemes. Served at `/.well-known/agent.json` for standard discovery. |
| **apcore** | Schema-driven module development framework. Protocol-agnostic core with adapter-based protocol support (MCP via apcore-mcp, A2A via apcore-a2a). |
| **Artifact** | Output produced by an A2A task. Contains Parts (TextPart, DataPart, FilePart) representing the result of module execution. |
| **contextId** | UUID v4 identifier grouping multiple messages into a multi-turn conversation. Enables stateful interactions and input elicitation. |
| **Executor** | apcore component that executes modules through the full pipeline: context creation, safety checks, ACL enforcement, input validation, middleware, execution, output validation. |
| **JSON-RPC 2.0** | Lightweight remote procedure call protocol using JSON encoding. Primary protocol binding for A2A communication. |
| **MCP** | Model Context Protocol. Anthropic's protocol for model-to-tool communication. Complementary to A2A: MCP connects AI assistants to tools; A2A connects agents to agents. |
| **Module** | An apcore capability unit with defined `input_schema`, `output_schema`, `description`, `annotations`, `tags`, and `examples`. The atomic unit of functionality in the apcore ecosystem. |
| **Part** | A piece of content within an A2A Message or Artifact. Types: TextPart (plain text), DataPart (structured data with media type), FilePart (file reference). |
| **Push Notification** | Webhook-based delivery of task state updates via HTTP POST. Third A2A interaction mode alongside synchronous and streaming. |
| **Registry** | apcore component that discovers, loads, and indexes modules. Provides `list()`, `get_definition()`, and event-based registration notifications. |
| **Skill** | An A2A concept representing a specific capability of an agent. Maps 1:1 to apcore modules in apcore-a2a. |
| **SSE** | Server-Sent Events. HTTP-based unidirectional streaming mechanism used for A2A `message/stream` to deliver `TaskStatusUpdateEvent` and `TaskArtifactUpdateEvent`. |
| **Task** | Stateful A2A work unit with lifecycle: submitted, working, completed, failed, canceled, input_required. Core concept for managing asynchronous and long-running operations. |
| **TaskStore** | Pluggable storage backend for persisting task state. In-memory default; can be replaced with Redis, PostgreSQL, or any custom implementation conforming to the `TaskStore` interface. |
| **xxx-apcore** | Convention for apcore adapter projects targeting specific domains: `comfyui-apcore`, `vnpy-apcore`, `blender-apcore`, etc. Each gains A2A capability by adding apcore-a2a as a dependency. |

### Appendix B: A2A Protocol Reference

**A2A v0.3.0 JSON-RPC Methods:**

| Method | Parameters | Response | Description |
|--------|-----------|----------|-------------|
| `message/send` | `{message, metadata, contextId?}` | Task | Synchronous message execution |
| `message/stream` | `{message, metadata, contextId?}` | SSE stream | Streaming execution |
| `tasks/get` | `{id}` | Task | Retrieve task by ID |
| `tasks/cancel` | `{id}` | Task | Cancel in-flight task |
| `tasks/resubscribe` | `{id}` | SSE stream | Reconnect to task stream |
| `tasks/pushNotificationConfig/set` | `{taskId, url}` | PushNotificationConfig | Register webhook |
| `tasks/pushNotificationConfig/get` | `{taskId}` | PushNotificationConfig | Get webhook config |
| `tasks/pushNotificationConfig/delete` | `{taskId}` | void | Remove webhook |

**A2A Error Codes:**

| Code | Name | Description |
|------|------|-------------|
| -32700 | Parse error | Invalid JSON |
| -32600 | Invalid Request | Invalid JSON-RPC structure |
| -32601 | Method not found | Unknown method or skill |
| -32602 | Invalid params | Missing or invalid parameters |
| -32603 | Internal error | Server-side execution error |
| -32001 | TaskNotFound | Task ID does not exist |
| -32002 | TaskNotCancelable | Task is in terminal state |

### Appendix C: apcore API Surface Reference

**Registry API:**
```python
registry = Registry(extensions_dir="./extensions")
registry.discover() -> int
registry.list(tags=None, prefix=None) -> list[str]
registry.get(module_id) -> module | None
registry.get_definition(module_id) -> ModuleDescriptor | None
registry.iter() -> Iterator[tuple[str, Any]]
registry.on("register", callback)
registry.on("unregister", callback)
```

**Executor API:**
```python
executor = Executor(registry, middlewares=None, acl=None, config=None)
await executor.call_async(module_id, inputs, context=None) -> dict
async for chunk in executor.stream(module_id, inputs, context=None): ...
executor.validate(module_id, inputs) -> ValidationResult
```

**Error Hierarchy:**
```python
ModuleError (base)
    +-- ModuleNotFoundError(module_id)
    +-- SchemaValidationError(message, errors)
    +-- ACLDeniedError(caller_id, target_id)
    +-- ModuleExecuteError(message)
    +-- ModuleTimeoutError(module_id, timeout_ms)
    +-- InvalidInputError(message)
    +-- CallDepthExceededError(depth, max_depth, call_chain)
    +-- CircularCallError(module_id, call_chain)
    +-- CallFrequencyExceededError(module_id, count, max_repeat, call_chain)
    +-- ApprovalPendingError(module_id)
```

### Appendix D: Schema Mapping Reference

#### D.1 apcore Registry to A2A Agent Card

```
apcore Registry                       A2A Agent Card
========================              ========================
(config) project.name           -->   AgentCard.name
(config) project.description    -->   AgentCard.description
(config) project.version        -->   AgentCard.version
(serve url)                     -->   AgentCard.url
(computed from modules)         -->   AgentCard.skills[]
(computed from annotations)     -->   AgentCard.capabilities
(auth config)                   -->   AgentCard.securitySchemes[]
(computed)                      -->   AgentCard.defaultInputModes
(computed)                      -->   AgentCard.defaultOutputModes
```

#### D.2 apcore Module to A2A Skill

```
apcore Module                         A2A Skill
========================              ========================
module_id                       -->   Skill.id
module_id (humanized)           -->   Skill.name
description                     -->   Skill.description
tags                            -->   Skill.tags
examples[].title                -->   Skill.examples[].name
examples[].input                -->   Skill.examples[].input (TextPart)
input_schema (JSON Schema)      -->   inputModes: ["application/json"]
output_schema (JSON Schema)     -->   outputModes: ["application/json"]
```

#### D.3 apcore Errors to A2A JSON-RPC Errors

```
apcore Error                          A2A JSON-RPC Error
========================              ========================
ModuleNotFoundError             -->   -32601 (Method not found)
SchemaValidationError           -->   -32602 (Invalid params)
ACLDeniedError                  -->   -32001 (TaskNotFound, hide details)
ModuleExecuteError              -->   -32603 (Internal error)
ModuleTimeoutError              -->   -32603 (Internal error)
InvalidInputError               -->   -32602 (Invalid params)
CallDepthExceededError          -->   -32603 (Internal error, safety limit)
CircularCallError               -->   -32603 (Internal error, safety limit)
CallFrequencyExceededError      -->   -32603 (Internal error, safety limit)
ApprovalPendingError            -->   NOT error; input_required state
Unknown exceptions              -->   -32603 (Internal error, generic)
```

#### D.4 apcore Execution to A2A Task Lifecycle

```
apcore Executor                       A2A Task
========================              ========================
executor.call_async() start     -->   status.state = "submitted"
module.execute() running        -->   status.state = "working"
executor.call_async() success   -->   status.state = "completed"
executor.call_async() error     -->   status.state = "failed"
ApprovalPendingError            -->   status.state = "input_required"
CancelToken.cancel()            -->   status.state = "canceled"
executor.stream() chunk         -->   TaskArtifactUpdateEvent (append: true)
executor.stream() complete      -->   TaskStatusUpdateEvent (completed)
```

---

*End of Document*
