# Product Requirements Document: apcore-a2a

| Field            | Value                                                                  |
|------------------|------------------------------------------------------------------------|
| **Title**        | apcore-a2a: Automatic A2A Protocol Adapter for apcore Module Registry  |
| **Version**      | 1.0                                                                    |
| **Date**         | 2026-03-03                                                             |
| **Author**       | aipartnerup Product Team                                               |
| **Status**       | Draft                                                                  |
| **Reviewers**    | apcore Core Maintainers, apcore-mcp Maintainers, Community Contributors |
| **Stakeholders** | apcore Core Team, A2A SDK Maintainers, xxx-apcore Project Developers, Enterprise Integration Partners |
| **License**      | Apache 2.0                                                             |

---

## Background

apcore-a2a originated from the observation that apcore modules already carry all the metadata the A2A (Agent-to-Agent) protocol needs -- `input_schema`, `output_schema`, `description`, `annotations`, `tags`, and `examples` -- yet there is no standard way to expose them to A2A clients, orchestrators, or peer agents. The idea was validated through analysis of the A2A protocol specification (v0.3.0), competitive landscape assessment (LangChain and CrewAI shipping A2A integrations), and demand signals (A2A adopted by 50+ partners including Google, Salesforce, SAP, ServiceNow; moved to Linux Foundation governance in April 2025). The core insight: since the mapping from apcore metadata to A2A concepts (Agent Card, Skills, Task lifecycle) is nearly 1:1, a single adapter package can eliminate all manual A2A server construction work for the entire apcore ecosystem. This is the natural counterpart to apcore-mcp: MCP handles model-to-tool communication; A2A handles agent-to-agent communication. Together they make apcore the only schema-driven framework that natively bridges to both major AI interoperability protocols from a single module definition. This PRD formalizes the validated idea into actionable requirements.

> For the original brainstorming and validation notes, see [`ideas/apcore-a2a.md`](../../ideas/apcore-a2a.md).

---

## 1. Executive Summary

**apcore-a2a** is an independent Python adapter package that automatically converts any apcore Module Registry into a fully functional A2A (Agent-to-Agent) protocol compliant agent server and client. Instead of requiring developers to hand-write Agent Cards, JSON-RPC endpoints, task state machines, and SSE streaming logic for every deployment, apcore-a2a reads the existing apcore metadata -- `input_schema`, `output_schema`, `description`, `annotations`, `tags`, and `examples` -- and generates a standards-compliant A2A agent at runtime with zero manual effort. The package exposes a primary entry point `serve(registry)` to launch an A2A server with automatic Agent Card generation, skill mapping, and full task lifecycle management, plus an `A2AClient` class for discovering and invoking remote A2A agents. By eliminating the 500+ lines of boilerplate required for manual A2A server construction, apcore-a2a completes the apcore ecosystem's protocol surface area -- with apcore-mcp handling model-to-tool (MCP) and apcore-a2a handling agent-to-agent (A2A) -- proving apcore is a universal framework for AI interoperability.

**Key business metrics impacted:** Ecosystem completeness (dual-protocol coverage from single module definition), developer adoption rate across multi-agent use cases, and number of downstream `xxx-apcore` projects with A2A agent capability.

---

## 2. Problem Statement

### 2.1 Current Pain Point

apcore modules are designed to be schema-driven and AI-perceivable. Every module carries `input_schema` (JSON Schema via Pydantic), `output_schema`, a human-readable `description`, behavioral `annotations` (readonly, destructive, idempotent, requires_approval, open_world), `tags`, and `examples`. This metadata is exactly what the A2A protocol requires to define agent capabilities as Skills and construct Agent Cards.

However, there is currently **no standard way to expose apcore modules as A2A agents**. A developer who wants their apcore modules discoverable and callable by other AI agents must:

1. Manually author an A2A Agent Card JSON document -- mapping module names to Skills, computing capabilities from annotations, declaring security schemes.
2. Stand up an HTTP server with JSON-RPC 2.0 request handling for `message/send`, `message/stream`, `tasks/get`, `tasks/cancel`, and all other A2A methods.
3. Implement the A2A task state machine (submitted, working, completed, failed, canceled, input_required) with thread-safe transitions, history tracking, and storage.
4. Build SSE streaming infrastructure for `message/stream` with `TaskStatusUpdateEvent` and `TaskArtifactUpdateEvent` events.
5. Wire execution routing from A2A message Parts to apcore Executor calls, preserving ACL, validation, and middleware.
6. Map apcore's error hierarchy to A2A JSON-RPC error codes with security-aware error sanitization.
7. Optionally implement push notification webhook delivery, multi-turn context management, and authentication bridging.

**Concrete example:** Building a compliant A2A server from scratch requires realistically 500+ lines of infrastructure code and 2-4 weeks of engineering effort per deployment, before any domain logic is written.

### 2.2 Evidence

| Pain Point | Evidence | Impact |
|------------|----------|--------|
| **No agent-to-agent interoperability** | apcore modules are only accessible via MCP; no A2A endpoint exists | Modules invisible to 50+ A2A partner ecosystem (Google, Salesforce, SAP, ServiceNow) |
| **Manual A2A implementation is costly** | Building an A2A server from scratch requires Agent Card authoring, JSON-RPC endpoints, task state machine, SSE streaming, push notifications | 500+ lines boilerplate; 2-4 weeks effort; error-prone lifecycle management |
| **No schema reuse** | Developers using A2A SDK directly must re-describe capabilities already defined in apcore modules | Duplicate work; schema drift between apcore definitions and A2A declarations |
| **Framework lock-in risk** | LangChain and CrewAI shipping A2A integrations within their ecosystems | Developers needing A2A may migrate to competing frameworks |
| **xxx-apcore projects half-connected** | comfyui-apcore, vnpy-apcore can serve MCP clients but not peer agents | Cannot participate in enterprise multi-agent workflows requiring A2A |

### 2.3 Root Cause

apcore was designed as a protocol-agnostic module framework, deliberately excluding protocol adapters from its core (as defined in SCOPE.md: "MCP/A2A Adapters --> Won't Do --> Independent adapter projects"). This is the correct architectural boundary. The missing piece is the independent adapter package that bridges apcore to A2A, following the same pattern established by apcore-mcp.

---

## 3. Product Vision

### 3.1 Vision Statement

Make every apcore module a first-class citizen in the agent-to-agent ecosystem. Any developer who builds an apcore module should automatically gain the ability to expose it as an A2A-discoverable skill -- and any agent orchestrator should be able to discover, invoke, and coordinate apcore-powered agents alongside agents built on any other framework. Combined with apcore-mcp, this establishes apcore as the only schema-driven framework with native dual-protocol coverage (MCP + A2A) from a single module definition.

### 3.2 Strategic Alignment with apcore Ecosystem

apcore-a2a fits precisely into the architecture defined by apcore's own SCOPE.md, which explicitly designates protocol adapters as independent projects:

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

**Key strategic pillars:**

- **Ecosystem Completeness**: MCP (model-to-tool) + A2A (agent-to-agent) = complete protocol surface. apcore becomes the only schema-driven framework bridging both major AI interoperability protocols.
- **Universal Framework Thesis**: apcore's core value proposition is "define once, deploy everywhere." apcore-a2a is the second proof point (after apcore-mcp) that apcore metadata is sufficient to auto-generate protocol surfaces.
- **Network Effects**: Every xxx-apcore project (comfyui-apcore, vnpy-apcore, blender-apcore, etc.) automatically gains A2A capability via a single dependency addition. This multiplies the value of the entire ecosystem.
- **Enterprise Readiness**: A2A is the protocol of choice for enterprise multi-agent deployments (backed by Google, Linux Foundation, 50+ partners). Supporting A2A positions apcore for enterprise adoption.
- **Proven Pattern**: apcore-mcp-python (v0.7.0, ~450 tests, 90% coverage) validates the adapter architecture. apcore-a2a follows the identical layered pattern with A2A-specific internals.

---

## 4. Target Users & Personas

### 4.1 P0 Personas (Primary -- Equal Priority)

#### Persona 1: Module Developer ("Maya")

| Attribute        | Detail                                                                 |
|------------------|------------------------------------------------------------------------|
| **Name**         | Maya                                                                   |
| **Role**         | Backend developer building apcore modules for domain-specific tools    |
| **Experience**   | 3-5 years Python; familiar with apcore-python; has built 5-15 modules  |
| **Demographics** | Individual contributor at mid-size tech company or open-source project |
| **Goal**         | Expose existing apcore modules as A2A-discoverable agents without learning A2A protocol internals |
| **Needs**        | One-call server startup; automatic Agent Card generation; zero A2A boilerplate; streaming support for long-running modules |
| **Pain Points**  | Spent 3 weeks manually building an A2A server for 10 modules; Agent Card JSON schema is tedious to author by hand; task lifecycle bugs consumed 40% of development time |
| **Success Metric** | Can serve a Registry of 20 modules as an A2A agent in under 5 minutes with zero lines of A2A-specific code |

**User journey (before):**
1. Maya writes apcore modules with `input_schema`, `output_schema`, `description`.
2. Maya wants other AI agents to discover and call her modules.
3. Maya reads A2A protocol specification (4+ hours).
4. Maya hand-writes Agent Card JSON, JSON-RPC handlers, task state machine, and SSE streaming (~2-4 weeks).
5. Maya discovers state transition bugs in multi-turn scenarios. Debugging takes 1+ week.
6. Maya gives up on push notifications because the webhook delivery logic adds another week.

**User journey (after):**
1. Maya writes apcore modules (same as before).
2. Maya runs `pip install apcore-a2a`.
3. Maya adds 3 lines of code: import, create registry, call `serve(registry)`.
4. Agent Card is automatically served at `/.well-known/agent.json`. All modules appear as Skills.
5. Other agents discover and invoke Maya's modules via standard A2A protocol.
6. When Maya updates a module schema, the Agent Card automatically reflects the change on next restart.

#### Persona 2: Agent Orchestrator ("Omar")

| Attribute        | Detail                                                                 |
|------------------|------------------------------------------------------------------------|
| **Name**         | Omar                                                                   |
| **Role**         | Platform engineer building multi-agent workflows that coordinate multiple AI agents |
| **Experience**   | 5+ years; works with LangChain, CrewAI, custom orchestrators; familiar with A2A protocol |
| **Demographics** | Tech lead at enterprise or late-stage startup; responsible for agent infrastructure |
| **Goal**         | Discover and invoke apcore-powered agents from orchestration pipelines; integrate apcore agents alongside non-apcore A2A agents seamlessly |
| **Needs**        | Standards-compliant Agent Card; reliable task lifecycle; SSE streaming for long-running operations; push notifications for async workflows; A2A Client for calling remote agents |
| **Pain Points**  | Cannot discover apcore agents via standard A2A discovery; has to build custom integration for each apcore deployment; no streaming support means polling overhead for long operations |
| **Success Metric** | Can discover and invoke apcore agents identically to any other A2A agent; multi-turn conversations work reliably across 10+ turns |

### 4.2 P1 Personas (Secondary)

#### Persona 3: xxx-apcore Project Developer ("Priya")

| Attribute        | Detail                                                                 |
|------------------|------------------------------------------------------------------------|
| **Name**         | Priya                                                                  |
| **Role**         | Developer maintaining a domain-specific apcore bridge (e.g., comfyui-apcore, vnpy-apcore) |
| **Experience**   | Familiar with both apcore and the domain tool; moderate Python experience |
| **Goal**         | Add A2A capability to their bridge project with minimal additional code and maintenance burden |
| **Needs**        | Drop-in `serve()` call; no domain-specific A2A configuration needed; all module metadata automatically inherited |
| **Pain Points**  | Currently can only offer MCP access; users requesting A2A support cannot be served; does not have bandwidth to maintain a custom A2A server alongside domain logic |
| **Success Metric** | All domain modules exposed via A2A with zero A2A-specific code in the bridge project |

#### Persona 4: Enterprise Platform Architect ("Elena")

| Attribute        | Detail                                                                 |
|------------------|------------------------------------------------------------------------|
| **Name**         | Elena                                                                  |
| **Role**         | Architect designing multi-agent systems for enterprise workflows       |
| **Experience**   | 10+ years; evaluates frameworks for security, compliance, scalability  |
| **Goal**         | Ensure apcore agents meet enterprise security requirements (OAuth 2.0, JWT, mTLS) and can participate in governed multi-agent topologies |
| **Needs**        | Security scheme declarations in Agent Card; authentication bridging to internal identity systems; audit trail via task history; compliance documentation |
| **Pain Points**  | Most agent frameworks lack proper security scheme declaration; A2A security configuration is complex and poorly documented |
| **Success Metric** | apcore-a2a passes enterprise security review; authenticated agent access works end-to-end with corporate OAuth 2.0 provider |

---

## 5. Market Analysis

### 5.1 Market Sizing

**Basis**: The multi-agent AI systems market is in early-growth phase (2025-2026). Estimates are grounded in the observable ecosystem of AI agent framework users, A2A protocol adopters, and apcore's addressable audience. No figures are fabricated; all reasoning is shown.

| Segment | Estimate | Reasoning |
|---------|----------|-----------|
| **TAM** (Total Addressable Market) | ~500,000 developers building multi-agent AI systems globally | Based on: ~2M AI/ML developers worldwide (GitHub Octoverse 2025 data), ~25% working on agent/agentic systems (growing rapidly), ~40% YoY growth in multi-agent tooling adoption |
| **SAM** (Serviceable Addressable Market) | ~50,000 developers using Python-based agent frameworks with A2A awareness | Based on: A2A protocol adopted by 50+ major partners; Python accounts for ~70% of AI development; subset actively building interoperable multi-agent systems |
| **SOM** (Serviceable Obtainable Market) | ~2,000-5,000 developers in the apcore ecosystem within 18 months | Based on: apcore-mcp current traction as baseline; compound growth from xxx-apcore projects adding A2A support; enterprise pilot conversions from dual-protocol demonstration |

**Key market signals:**
- A2A moved to Linux Foundation governance (April 2025), signaling long-term institutional backing and stability.
- Multi-agent architectures are the dominant pattern for complex AI workflows in 2025-2026.
- Enterprise demand for agent interoperability is the #1 cited need in AI platform surveys.
- A2A partner list includes Google, Atlassian, Salesforce, SAP, ServiceNow, LangChain, and 50+ others.

### 5.2 Competitive Landscape

| Solution | Approach | A2A Compliance | Schema Reuse | Setup Effort | Streaming | Push Notifications | Multi-Turn | Ecosystem Lock-in |
|----------|----------|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| **Manual A2A Server** | Hand-write Agent Card, JSON-RPC, task lifecycle | Full (developer-controlled) | None -- schemas defined from scratch | High (500+ lines, 2-4 weeks) | Manual implementation | Manual implementation | Manual implementation | None |
| **A2A SDK Direct** | Build agent from scratch using official SDK | Full | None -- capabilities re-described | Medium (200+ lines, 1-2 weeks) | SDK-assisted | SDK-assisted | Manual implementation | None |
| **LangChain A2A** | LangChain agent exposed via A2A wrapper | Partial (LangChain-filtered) | LangChain tools only | Low-Medium | LangChain streaming | Not yet available | LangChain memory | High (LangChain) |
| **CrewAI A2A** | CrewAI agent exposed via A2A bridge | Partial | CrewAI tasks only | Low-Medium | Limited | Not yet available | CrewAI context | High (CrewAI) |
| **apcore-a2a (this project)** | Auto-bridge from apcore Registry | **Full (v0.3.0)** | **Complete** (schema, annotations, tags, examples) | **Very Low** (1 function call) | **Automatic** (modules with streaming) | **Included** (v1) | **contextId-based** | Moderate (apcore) |

### 5.3 Differentiation Summary

apcore-a2a's unique advantages:

1. **Complete schema reuse**: apcore modules already carry the exact metadata A2A needs (input_schema, output_schema, description, annotations, tags, examples). No other solution achieves zero-duplication mapping from module definitions to A2A Skills and Agent Cards.
2. **Dual-protocol from single source**: The same apcore module definition serves both MCP and A2A protocols. No competing framework offers this -- it is a structural advantage from apcore's schema-driven design.
3. **Annotation-driven capabilities**: apcore annotations automatically inform A2A Agent Card `capabilities`, Skill metadata, and task lifecycle behavior. Competing solutions require manual capability declaration.
4. **Proven adapter pattern**: apcore-mcp-python (v0.7.0, ~450 tests, 90% coverage) validates the adapter architecture. apcore-a2a follows the identical layered pattern (adapters/, server/, auth/) with A2A-specific internals.
5. **Full v1 coverage**: Synchronous, streaming, and push notification interaction modes are all included in v1. Most competing solutions ship sync-only or defer push notifications indefinitely.

---

## 6. User Stories & Use Cases

### 6.1 User Stories

#### US-001: One-Call Server Startup

**As a** module developer,
**I want to** start an A2A server with a single `serve(registry)` call,
**So that** I can expose my modules as A2A agents without writing protocol-level code.

**Acceptance Criteria:**
- `serve(registry)` starts an HTTP server on the default port (configurable via `host` and `port` parameters).
- The server responds to `GET /.well-known/agent.json` with a valid A2A Agent Card.
- All registered modules appear as Skills in the Agent Card.
- The server handles JSON-RPC 2.0 requests at the root endpoint.
- The server shuts down cleanly on SIGINT/SIGTERM.
- Calling `serve(registry)` with an empty registry raises `ValueError` with a descriptive message.

#### US-002: Automatic Agent Card Discovery

**As an** agent orchestrator,
**I want to** discover an apcore agent's capabilities via the standard Agent Card endpoint,
**So that** I can integrate it into my multi-agent workflow like any other A2A agent.

**Acceptance Criteria:**
- `GET /.well-known/agent.json` returns a JSON document conforming to A2A v0.3.0 Agent Card schema.
- The Agent Card includes `name`, `description`, `version`, `url`, `skills[]`, `capabilities`.
- Each Skill includes `id`, `description`, `tags`, and `examples` derived from apcore module metadata.
- `capabilities.streaming` is `true` if any module supports streaming.
- `capabilities.pushNotifications` is `true` if push notification support is configured.
- `capabilities.stateTransitionHistory` is `true` if the task store supports history.

#### US-003: Synchronous Task Execution

**As an** agent orchestrator,
**I want to** send a `message/send` request and receive the completed result synchronously,
**So that** I can invoke apcore modules from my agent without managing streaming connections.

**Acceptance Criteria:**
- `message/send` JSON-RPC request routes to the correct apcore module based on skill targeting.
- The response contains a Task object with status `completed` and module output as an Artifact.
- If the module fails, the response contains a Task with status `failed` and an error message.
- Input validation errors return JSON-RPC error code -32602 (Invalid params) with field-level details.
- The full apcore Executor pipeline is invoked (ACL, validation, middleware).

#### US-004: Streaming Execution

**As an** agent orchestrator,
**I want to** send a `message/stream` request and receive incremental updates via SSE,
**So that** I can display progress and partial results for long-running operations.

**Acceptance Criteria:**
- `message/stream` returns an SSE stream with `Content-Type: text/event-stream`.
- `TaskStatusUpdateEvent` is sent on each state transition (submitted, working, completed, failed).
- `TaskArtifactUpdateEvent` is sent when the module produces incremental output.
- The stream terminates with a final `TaskStatusUpdateEvent` (completed or failed).
- Non-streaming modules fall back to synchronous execution with status-only updates.

#### US-005: Task Lifecycle Management

**As an** agent orchestrator,
**I want to** query, list, and cancel tasks,
**So that** I can manage long-running operations, implement retry logic, and clean up stale tasks.

**Acceptance Criteria:**
- `tasks/get` returns the current state of a task by ID.
- `tasks/list` returns tasks, optionally filtered by `contextId`.
- `tasks/cancel` transitions a cancelable task to `canceled` state.
- Canceling a non-cancelable task returns error code -32002 (TaskNotCancelable).
- Task state transitions follow the A2A state machine (no invalid transitions).

#### US-006: Multi-Turn Conversations

**As an** agent orchestrator,
**I want to** send multiple messages within the same conversation context,
**So that** I can build interactive workflows that maintain state across turns and handle input elicitation.

**Acceptance Criteria:**
- Messages with the same `contextId` are grouped into a conversation.
- Previous messages in the context are accessible to the module during execution.
- A new `contextId` starts a fresh conversation.
- The `input_required` state allows the orchestrator to provide additional input before resuming.
- `tasks/list` can filter by `contextId` to retrieve all tasks in a conversation.

#### US-007: Remote Agent Discovery and Invocation

**As a** module developer building an agent that collaborates with other agents,
**I want to** discover and call remote A2A agents using a Python client,
**So that** my apcore agent can participate in multi-agent workflows as both a server and a client.

**Acceptance Criteria:**
- `A2AClient(url)` constructs a client pointing to a remote agent's base URL.
- `client.agent_card` fetches and caches the Agent Card from `<url>/.well-known/agent.json`.
- `client.send_message(message)` sends `message/send` and returns the Task.
- `client.stream_message(message)` sends `message/stream` and returns an async iterator of events.
- `client.get_task(task_id)` retrieves task status.
- `client.cancel_task(task_id)` cancels a remote task.
- The client handles A2A error codes correctly and raises typed exceptions.

#### US-008: Push Notification Integration

**As an** agent orchestrator running asynchronous workflows,
**I want to** register a webhook URL and receive push notifications when task state changes,
**So that** I do not have to poll for task completion in long-running scenarios.

**Acceptance Criteria:**
- `tasks/pushNotificationConfig/set` registers a webhook URL for a task.
- The server sends HTTP POST to the webhook URL on each task state transition.
- `tasks/pushNotificationConfig/get` returns the current push notification configuration.
- `tasks/pushNotificationConfig/delete` removes the webhook.
- Failed webhook delivery retries up to 3 times with exponential backoff.

#### US-009: Authenticated Agent Access

**As an** enterprise platform architect,
**I want to** secure the A2A server with JWT/Bearer authentication,
**So that** only authorized agents can invoke my modules and the caller's identity flows through to apcore's ACL system.

**Acceptance Criteria:**
- Agent Card declares supported security schemes (Bearer, OAuth 2.0).
- Incoming requests with valid JWT tokens are authenticated.
- The authenticated identity is bridged to apcore's Identity context variable.
- Invalid/missing tokens return HTTP 401 Unauthorized.
- ACL-denied module calls return A2A error with details hidden per security policy.

#### US-010: CLI Entry Point

**As a** module developer,
**I want to** start an A2A server from the command line,
**So that** I can quickly test and deploy without writing Python code.

**Acceptance Criteria:**
- `apcore-a2a serve --extensions-dir ./extensions` starts the server.
- CLI supports `--host`, `--port`, `--name`, `--description`, `--version` options.
- `apcore-a2a --help` displays usage information.
- Exit codes follow standard conventions (0 = success, 1 = error).

### 6.2 Use Cases

#### Use Case 1: Module Developer Deploys A2A Agent

```
Actor: Module Developer (Maya)
Precondition: apcore Registry with 10+ modules is ready

Flow:
  1. Maya installs apcore-a2a:
       pip install apcore-a2a
  2. Maya adds 3 lines to her application:
       from apcore import Registry
       from apcore_a2a import serve
       serve(Registry(extensions_dir="./extensions"))
  3. Server starts on port 8000 (default)
  4. Maya verifies Agent Card:
       curl http://localhost:8000/.well-known/agent.json
  5. Agent Card returns JSON with all 10 modules listed as Skills
  6. Maya tests a synchronous call:
       curl -X POST http://localhost:8000/ \
         -H "Content-Type: application/json" \
         -d '{"jsonrpc":"2.0","id":"1","method":"message/send",
              "params":{"message":{"role":"user",
              "parts":[{"type":"text","text":"..."}]},
              "metadata":{"skillId":"image.resize"}}}'
  7. Response contains completed Task with module output as Artifact

Postcondition: All 10 modules are discoverable and callable via A2A protocol

Alternative Flow:
  4a. Agent Card is missing a module
      -> Maya checks Registry.list() to confirm module is registered
      -> Module with missing description is excluded with a warning log
```

#### Use Case 2: Orchestrator Integrates apcore Agent into Multi-Agent Workflow

```
Actor: Agent Orchestrator (Omar)
Precondition: Multi-agent workflow with 3 agents (2 non-apcore, 1 apcore)

Flow:
  1. Omar's orchestrator discovers agents via Agent Card URLs
  2. Orchestrator fetches apcore agent's Agent Card at
       GET https://agent.example.com/.well-known/agent.json
  3. Orchestrator parses Skills array and selects relevant capabilities
  4. Orchestrator sends message/send to apcore agent for data processing:
       {"method":"message/send","params":{
         "message":{"role":"user","parts":[...]},
         "metadata":{"skillId":"data.transform"}}}
  5. apcore agent returns completed Task with Artifact
  6. Orchestrator passes Artifact content to next agent in pipeline
  7. Orchestrator uses message/stream for a long-running apcore module
  8. SSE events stream:
       - TaskStatusUpdateEvent (state: "submitted")
       - TaskStatusUpdateEvent (state: "working")
       - TaskArtifactUpdateEvent (partial results)
       - TaskStatusUpdateEvent (state: "completed")
  9. Workflow completes with coordinated results from all 3 agents

Postcondition: apcore agent participated identically to non-apcore A2A agents

Alternative Flow:
  4a. Module requires approval (requires_approval annotation)
      -> Task transitions to input_required state
      -> Orchestrator sends follow-up message with approval in same contextId
      -> Task resumes and completes
```

#### Use Case 3: Enterprise Multi-Agent System with Security

```
Actor: Enterprise Platform Architect (Elena)
Precondition: Enterprise environment with OAuth 2.0 identity provider

Flow:
  1. Elena configures apcore-a2a with JWT authentication:
       serve(registry, auth=JWTAuthenticator(
           key="...",
           issuer="https://idp.corp.com",
           audience="apcore-agents"))
  2. Agent Card at /.well-known/agent.json declares:
       "securitySchemes": [{"type": "http", "scheme": "bearer"}]
  3. Extended Agent Card at /agent/authenticatedExtendedCard
       includes additional privileged Skills not on public card
  4. Omar's orchestrator authenticates via corporate OAuth 2.0 flow
  5. Orchestrator sends message/send with Bearer token header
  6. apcore-a2a validates JWT, extracts identity claims (sub, roles)
  7. Identity is bridged to apcore's Identity context variable
  8. Module executes with ACL checks against the authenticated identity
  9. If ACL permits: Task completes normally with Artifact
  10. Audit trail: task history records identity and state transitions

Postcondition: Only authorized agents can invoke protected modules;
               identity flows through to apcore ACL system

Alternative Flow:
  5a. Token is expired or invalid
      -> Server returns HTTP 401 Unauthorized
      -> Orchestrator refreshes token and retries
  9a. ACL denies access
      -> Task fails with generic error message (no security info leak)
```

---

## 7. Feature Requirements

### P0 -- Must Have (Launch Blockers)

---

#### FR-001: serve() -- One-Call A2A Server from Registry

**Title:** Primary entry point for launching an A2A-compliant agent server

**Description:** Provide a top-level `serve()` function that accepts an apcore Registry (or Executor) and starts a fully functional A2A-compliant HTTP server. This is the primary public API and mirrors apcore-mcp's `serve(registry)` pattern. An `async_serve()` variant returns the ASGI app without starting uvicorn, for embedding in larger applications.

```python
from apcore import Registry
from apcore_a2a import serve

registry = Registry(extensions_dir="./extensions")
registry.discover()

# Simplest usage -- HTTP server on 0.0.0.0:8000
serve(registry)
```

**User Story:** US-001

**Acceptance Criteria:**
1. `serve(registry)` starts an HTTP server using uvicorn on `0.0.0.0:8000` by default.
2. `serve(registry, host="127.0.0.1", port=9000)` allows custom host and port configuration.
3. `serve(executor)` accepts an Executor instance (duck-typed: any object with `call_async()`, `stream()`, `validate()`).
4. Optional keyword arguments: `name`, `description`, `version`, `url`, `auth`, `task_store`, `cors_origins`, `push_notifications`, `explorer`, `explorer_prefix`, `cancel_on_disconnect`, `shutdown_timeout`, `execution_timeout`, `log_level`, `metrics`.
5. Server responds to A2A JSON-RPC 2.0 requests at the root path `/`.
6. Server responds to Agent Card requests at `GET /.well-known/agent.json`.
7. Server shuts down cleanly on SIGINT and SIGTERM signals.
8. Function raises `ValueError` if the registry/executor has zero modules.
9. `async_serve()` variant returns the ASGI application without starting uvicorn (for embedding in existing ASGI servers).
10. The function blocks until the server is shut down (consistent with server behavior).

**Dependencies:** FR-002, FR-003, FR-004, FR-005, FR-006, FR-007

**Priority:** P0

---

#### FR-002: Agent Card Auto-Generation from Registry Metadata

**Title:** Automatic A2A Agent Card construction from apcore Registry

**Description:** Automatically construct a valid A2A Agent Card from apcore Registry metadata. The Agent Card is the discovery mechanism at `/.well-known/agent.json` that allows other agents and orchestrators to understand this agent's capabilities, skills, and security requirements.

**Mapping:**

| apcore Source | A2A Agent Card Field |
|---------------|---------------------|
| `(config) project.name` | `AgentCard.name` (fallback: `"apcore-agent"`) |
| `(config) project.description` | `AgentCard.description` (fallback: auto-generated summary) |
| `(config) project.version` | `AgentCard.version` (fallback: `"0.0.0"`) |
| `(serve url)` | `AgentCard.url` |
| `(computed from modules)` | `AgentCard.skills[]` |
| `(computed from annotations)` | `AgentCard.capabilities` |
| `(auth config)` | `AgentCard.securitySchemes[]` |
| `(computed)` | `AgentCard.defaultInputModes` |
| `(computed)` | `AgentCard.defaultOutputModes` |

**User Story:** US-002

**Acceptance Criteria:**
1. Agent Card conforms to A2A v0.3.0 Agent Card JSON Schema.
2. `AgentCard.name` populated from Registry config `project.name`; fallback to `"apcore-agent"` if not configured.
3. `AgentCard.description` populated from Registry config `project.description`; fallback to auto-generated description listing module count.
4. `AgentCard.version` populated from Registry config `project.version`; fallback to `"0.0.0"`.
5. `AgentCard.url` populated from the server's bound address (scheme + host + port).
6. `AgentCard.skills[]` populated from module definitions (see FR-003).
7. `AgentCard.capabilities.streaming` set to `true` if any registered module supports streaming.
8. `AgentCard.capabilities.pushNotifications` set to `true` if push notifications are enabled via configuration.
9. `AgentCard.capabilities.stateTransitionHistory` set to `true` if the task store supports history recording.
10. `GET /.well-known/agent.json` returns HTTP 200 with `Content-Type: application/json`.
11. Agent Card is re-generated if modules are added/removed at runtime (when dynamic registration is enabled).

**Dependencies:** FR-003

**Priority:** P0

---

#### FR-003: Skill Mapping -- apcore Modules to A2A Skills

**Title:** Automatic conversion of apcore module definitions to A2A Skill objects

**Description:** Convert each apcore module definition into an A2A Skill object. This mapping is the core schema bridge that makes apcore modules discoverable via A2A Agent Cards.

**Mapping:**

| apcore Module Field | A2A Skill Field |
|--------------------|-----------------|
| `module_id` | `Skill.id` |
| `module_id` (humanized: underscores to spaces, title case) | `Skill.name` |
| `description` | `Skill.description` |
| `tags[]` | `Skill.tags[]` |
| `examples[].title` | `Skill.examples[].name` |
| `examples[].input` | `Skill.examples[].input` (as TextPart) |
| `input_schema` (JSON Schema) | `inputModes: ["application/json"]` |
| `output_schema` (JSON Schema) | `outputModes: ["application/json"]` |

**User Story:** US-002

**Acceptance Criteria:**
1. Each registered module produces exactly one Skill in the Agent Card.
2. `Skill.id` equals the apcore `module_id`.
3. `Skill.name` is derived from `module_id` by replacing dots and underscores with spaces and applying title case.
4. `Skill.description` equals the apcore module `description`.
5. `Skill.tags[]` equals the apcore module `tags[]`.
6. `Skill.examples[]` is generated from `module.examples[].title` and `module.examples[].input`.
7. Skills with `input_schema` declare `inputModes: ["application/json"]`.
8. Skills with `output_schema` declare `outputModes: ["application/json"]`.
9. Modules with text-type schemas additionally declare `text/plain` in input/output modes.
10. Modules with missing or empty `description` are excluded from Skills with a warning log.

**Dependencies:** None (foundational)

**Priority:** P0

---

#### FR-004: message/send -- Synchronous Message Handling

**Title:** JSON-RPC `message/send` method for synchronous task execution

**Description:** Implement the `message/send` JSON-RPC method that accepts a Message, routes it to the appropriate apcore module, executes it through the Executor pipeline, and returns the completed Task with Artifacts.

**User Story:** US-003

**Acceptance Criteria:**
1. Accepts JSON-RPC 2.0 request with method `message/send`.
2. Extracts target module from message `metadata.skillId` or equivalent skill targeting mechanism.
3. Creates a new Task with status `submitted` and unique `taskId` (UUID v4).
4. Transitions Task to `working` during module execution.
5. On successful execution, transitions Task to `completed` with module output as Artifact.
6. On execution error, transitions Task to `failed` with error message in Task status.
7. Returns the Task object in the JSON-RPC response.
8. Supports `contextId` for multi-turn conversation grouping (see FR-011).
9. If no target module can be determined from the message, returns JSON-RPC error -32601 (Method not found).
10. If the target module raises `ApprovalPendingError`, Task transitions to `input_required` state.

**Dependencies:** FR-003, FR-005, FR-006, FR-007

**Priority:** P0

---

#### FR-005: Task Lifecycle Management (State Machine)

**Title:** A2A task state machine with thread-safe transitions and history

**Description:** Implement the A2A task state machine that governs all task state transitions. This is the stateful core of the A2A server, ensuring only valid transitions occur and maintaining complete task history when configured.

**State diagram:**

```
                      +-- canceled
                      |
  submitted --> working --> completed
       |           |
       |           +-- failed
       |           |
       +-- canceled   +-- input_required --> working
       |                                       |
       +-- failed                              +-- canceled
                                               |
                                               +-- failed
```

**User Story:** US-005

**Acceptance Criteria:**
1. Task states: `submitted`, `working`, `completed`, `failed`, `canceled`, `input_required`.
2. Valid transitions enforced:
   - `submitted` --> `working`, `canceled`, `failed`
   - `working` --> `completed`, `failed`, `canceled`, `input_required`
   - `input_required` --> `working`, `canceled`, `failed`
3. Invalid state transitions raise an internal error (logged at ERROR level, not exposed to client).
4. Each Task object has: `id`, `contextId`, `status` (with `state`, `message`, `timestamp`), `artifacts[]`, `history[]`.
5. `status.timestamp` is updated on every state transition (ISO 8601 format).
6. `history[]` records all previous state transitions when `stateTransitionHistory` is enabled.
7. Task objects are stored in the configured task store (see FR-015; in-memory default).
8. Concurrent access to a single task is thread-safe (atomic state transitions).

**Dependencies:** FR-015 (soft dependency; in-memory store used by default)

**Priority:** P0

---

#### FR-006: Execution Routing -- A2A to apcore Executor Pipeline

**Title:** Route A2A messages through the full apcore Executor pipeline

**Description:** Route incoming A2A messages to the appropriate apcore module through the full Executor pipeline, preserving ACL enforcement, input validation, middleware execution, and timeout handling. This ensures the same security and quality guarantees apply regardless of whether a module is invoked via MCP, A2A, or direct API call.

**User Story:** US-003

**Acceptance Criteria:**
1. Message content is parsed to determine the target module and input parameters.
2. JSON-typed Parts (`application/json`) are deserialized to module input dicts.
3. Text-typed Parts are passed as string input where the module's `input_schema` accepts text.
4. Execution uses `Executor.call_async()` to ensure the full pipeline (ACL, validation, middleware, timeout).
5. Module output is converted to A2A Artifact Parts: JSON output --> DataPart; text output --> TextPart.
6. `Executor.validate()` is called before execution when input schema is defined.
7. Validation failures return JSON-RPC error -32602 (Invalid params) with field-level details.
8. Execution timeout is configurable (default: 300 seconds).
9. ACL context (`Identity`) is populated from the authenticated request when auth is configured (see FR-013).
10. If the module raises `ApprovalPendingError`, the task transitions to `input_required` state (not an error).

**Dependencies:** FR-005, FR-007

**Priority:** P0

---

#### FR-007: Error Mapping -- apcore Errors to A2A JSON-RPC Errors

**Title:** Deterministic mapping from apcore error hierarchy to A2A JSON-RPC error codes

**Description:** Map each apcore error type to an appropriate A2A JSON-RPC error response, following security best practices (no information leakage for access control errors, no stack trace exposure for internal errors).

**Mapping:**

| apcore Error | A2A JSON-RPC Error Code | Behavior |
|-------------|------------------------|----------|
| `ModuleNotFoundError` | -32601 (Method not found) | Module ID included in message |
| `SchemaValidationError` | -32602 (Invalid params) | Field-level validation details in `data` |
| `ACLDeniedError` | -32001 (TaskNotFound) | Details hidden per security policy |
| `ModuleExecuteError` | -32603 (Internal error) | Sanitized message; no stack trace |
| `ModuleTimeoutError` | -32603 (Internal error) | "Execution timed out" message |
| `ApprovalPendingError` | *Not an error* | Task transitions to `input_required` |
| `InvalidInputError` | -32602 (Invalid params) | Input error details in `data` |
| `CallDepthExceededError` | -32603 (Internal error) | "Safety limit exceeded" message |
| `CircularCallError` | -32603 (Internal error) | "Safety limit exceeded" message |
| Unknown exceptions | -32603 (Internal error) | Generic "Internal error" message |

**User Story:** US-003

**Acceptance Criteria:**
1. Each apcore error type listed above produces a distinct, identifiable JSON-RPC error response.
2. `SchemaValidationError` responses include field-level error details (field name, error code, message) in the `data` field.
3. `ACLDeniedError` responses do not leak sensitive security information (no caller_id, target_id, or ACL rule details).
4. Unexpected exceptions produce a generic "Internal error" message without leaking stack traces, file paths, or internal configuration.
5. All error responses include a `data` field with an error type identifier string for programmatic handling.
6. Error messages never contain internal file paths, stack traces, or sensitive configuration values.
7. `ApprovalPendingError` is handled as a task state transition to `input_required`, not as a JSON-RPC error.

**Dependencies:** None (foundational)

**Priority:** P0

---

#### FR-008: tasks/get, tasks/list, tasks/cancel

**Title:** Task query and management JSON-RPC methods

**Description:** Implement the task query and control JSON-RPC methods that allow clients to inspect task state, list tasks by context, and cancel in-flight operations.

**User Story:** US-005

**Acceptance Criteria:**
1. `tasks/get` with `{"id": "<taskId>"}` returns the full Task object including status, artifacts, and history.
2. `tasks/get` for a non-existent task ID returns error -32001 (TaskNotFound).
3. `tasks/list` returns all tasks, optionally filtered by `contextId` parameter.
4. `tasks/list` supports pagination via `cursor` and `limit` parameters (default limit: 50, max: 200).
5. `tasks/cancel` transitions a `submitted` or `working` task to `canceled` state.
6. `tasks/cancel` triggers cancellation of the underlying apcore execution via CancelToken.
7. `tasks/cancel` on a `completed`, `failed`, or already `canceled` task returns error -32002 (TaskNotCancelable).
8. All methods validate required parameters; missing params return -32602 (Invalid params).
9. Task data is served from the configured task store.

**Dependencies:** FR-005, FR-015

**Priority:** P0

---

#### FR-009: SSE Streaming -- message/stream

**Title:** Server-Sent Events streaming for real-time task progress and incremental results

**Description:** Implement the `message/stream` JSON-RPC method that returns an SSE stream for real-time task status updates and incremental artifact delivery. For streaming-capable modules, intermediate output chunks produce `TaskArtifactUpdateEvent` events; for non-streaming modules, only status update events are emitted.

**User Story:** US-004

**Acceptance Criteria:**
1. `message/stream` returns an HTTP response with `Content-Type: text/event-stream`.
2. `TaskStatusUpdateEvent` is emitted on each state transition (submitted --> working --> completed/failed).
3. `TaskArtifactUpdateEvent` is emitted when the module produces incremental output.
4. Events conform to A2A v0.3.0 SSE event schema.
5. For streaming-capable modules (`Executor.stream()`), intermediate output chunks produce `TaskArtifactUpdateEvent` with `append: true`.
6. For non-streaming modules, the stream emits status updates only (submitted --> working --> completed/failed).
7. The final event is always a `TaskStatusUpdateEvent` with a terminal state (completed, failed, or canceled).
8. Client disconnection terminates the stream and optionally cancels the task (configurable via `cancel_on_disconnect`).
9. `tasks/resubscribe` allows reconnecting to an active task's event stream.
10. SSE events include an `id` field for resumability.

**Dependencies:** FR-004, FR-005, FR-006

**Priority:** P0

---

#### FR-010: A2A Client -- Discover and Call Remote A2A Agents

**Title:** Python client for discovering and invoking remote A2A agents

**Description:** Provide an `A2AClient` class that allows apcore agents (and any Python application) to discover and invoke remote A2A-compliant agents. This enables apcore to participate in multi-agent workflows as both a server and a client. The client is usable independently of the server (standalone import without server-side dependencies).

```python
from apcore_a2a.client import A2AClient

client = A2AClient("https://remote-agent.example.com")

# Discover capabilities
card = await client.agent_card
print(card.skills)

# Send a synchronous message
task = await client.send_message(message)
print(task.status.state)  # "completed"

# Stream a long-running operation
async for event in client.stream_message(message):
    print(event)
```

**User Story:** US-007

**Acceptance Criteria:**
1. `A2AClient(url)` constructs a client pointing to a remote agent's base URL.
2. `client.agent_card` property fetches and caches the Agent Card from `<url>/.well-known/agent.json`.
3. `client.send_message(message)` sends `message/send` JSON-RPC request and returns the Task.
4. `client.stream_message(message)` sends `message/stream` and returns an async iterator of SSE events.
5. `client.get_task(task_id)` sends `tasks/get` and returns the Task.
6. `client.cancel_task(task_id)` sends `tasks/cancel`.
7. `client.list_tasks(context_id=None)` sends `tasks/list` with optional filter.
8. Client supports Bearer token authentication via `auth` parameter.
9. Client uses `httpx` for HTTP requests with configurable timeout (default: 30 seconds).
10. Client raises typed exceptions for A2A error codes (`TaskNotFoundError`, `TaskNotCancelableError`, etc.).
11. Agent Card cache has configurable TTL (default: 5 minutes).
12. Client is importable independently: `from apcore_a2a.client import A2AClient` does not import server-side dependencies.

**Dependencies:** None (independent module)

**Priority:** P0

---

#### FR-011: Multi-Turn Conversations via contextId

**Title:** Stateful multi-turn interactions grouped by conversation context

**Description:** Support stateful, multi-turn interactions where multiple messages are grouped under a shared `contextId`, enabling conversational workflows, input elicitation, and iterative refinement. This maps apcore's `requires_approval` annotation to A2A's `input_required` state for human-in-the-loop scenarios.

**User Story:** US-006

**Acceptance Criteria:**
1. `message/send` and `message/stream` accept an optional `contextId` parameter.
2. If `contextId` is provided, the message is added to the existing conversation context.
3. If `contextId` is not provided, a new `contextId` is generated (UUID v4).
4. Tasks within the same context can access previous messages (conversation history).
5. `input_required` state includes a message describing what additional input is needed.
6. Follow-up message with the same `contextId` resumes the `input_required` task back to `working`.
7. `tasks/list` can filter by `contextId` to retrieve all tasks in a conversation.
8. Context state is stored in the task store alongside tasks.
9. Context history is bounded (configurable max messages, default: 100).

**Dependencies:** FR-004, FR-005, FR-015

**Priority:** P0

---

### P1 -- Should Have (Important for User Satisfaction)

---

#### FR-012: Push Notifications (Webhook Delivery)

**Title:** Asynchronous task state updates via webhook push notifications

**Description:** Implement A2A push notification support, allowing clients to register webhook URLs and receive task state updates via HTTP POST callbacks. This enables the third A2A interaction mode (async push) in addition to synchronous and streaming.

**User Story:** US-008

**Acceptance Criteria:**
1. `tasks/pushNotificationConfig/set` registers a webhook URL for a specific task.
2. `tasks/pushNotificationConfig/get` returns the current push notification configuration for a task.
3. `tasks/pushNotificationConfig/list` returns all push notification configurations.
4. `tasks/pushNotificationConfig/delete` removes a push notification configuration.
5. On each task state transition, an HTTP POST is sent to the registered webhook URL with the event payload.
6. Webhook payload conforms to A2A v0.3.0 push notification schema.
7. Failed deliveries retry up to 3 times with exponential backoff (1s, 2s, 4s delays).
8. After 3 failed retries, the push notification configuration is marked as `failed`.
9. Webhook URLs must be HTTPS in production mode; HTTP allowed in development mode (configurable).
10. Agent Card `capabilities.pushNotifications` is set to `true` when push notifications are enabled.
11. Push notifications are disabled by default; enabled via `serve(registry, push_notifications=True)`.

**Dependencies:** FR-005, FR-008

**Priority:** P1

---

#### FR-013: JWT/Bearer Authentication Bridging to apcore Identity

**Title:** Authentication middleware bridging JWT tokens to apcore Identity context

**Description:** Implement authentication middleware that validates JWT/Bearer tokens on incoming A2A requests and bridges the authenticated identity to apcore's Identity context variable system, enabling apcore's ACL conditions (`identity_types`, `roles`) to work with external agent callers.

**User Story:** US-009

**Acceptance Criteria:**
1. `serve(registry, auth=JWTAuthenticator(key=..., issuer=..., audience=...))` enables JWT authentication.
2. Incoming requests with `Authorization: Bearer <token>` are validated (signature, expiration, issuer, audience).
3. JWT claims (`sub`, `email`, `name`, `roles`, custom claims) are extracted.
4. Extracted identity is set as apcore's Identity context variable via `ContextVar` bridge.
5. Modules can access the authenticated identity via standard apcore `context.identity` API.
6. Requests without tokens receive HTTP 401 response with `WWW-Authenticate: Bearer`.
7. Requests with invalid/expired tokens receive HTTP 401 response; token content is never leaked in error messages.
8. Agent Card `securitySchemes` array is populated based on auth configuration.
9. API Key authentication is also supported as an alternative scheme.
10. Auth middleware is pluggable: `Authenticator` is a `@runtime_checkable` Protocol allowing custom auth backends.

**Dependencies:** FR-001, FR-006

**Priority:** P1

---

#### FR-014: Extended Agent Card (Authenticated Endpoint)

**Title:** Authenticated Agent Card with additional privileged skills

**Description:** Serve an extended Agent Card at an authenticated endpoint that includes additional Skills not visible on the public Agent Card -- specifically modules with ACL restrictions or `requires_approval` annotations.

**User Story:** US-009

**Acceptance Criteria:**
1. `GET /agent/authenticatedExtendedCard` returns an extended Agent Card.
2. The extended card includes all public Skills plus Skills marked as requiring authentication.
3. The endpoint requires valid authentication; returns HTTP 401 without valid credentials.
4. Modules with `requires_approval` annotation or ACL restrictions appear only in the extended card.
5. `agentCard/get` JSON-RPC method returns the appropriate card based on authentication status.
6. The extended card declares all security schemes the agent supports.

**Dependencies:** FR-002, FR-013

**Priority:** P1

---

#### FR-015: Task Storage Interface (In-Memory Default, Pluggable)

**Title:** Pluggable task storage with default in-memory implementation

**Description:** Define an abstract task storage interface (`TaskStore`) with a default in-memory implementation. The interface supports pluggable persistent storage backends (Redis, PostgreSQL, etc.) for production deployments requiring task persistence across server restarts.

**User Story:** US-005

**Acceptance Criteria:**
1. `TaskStore` abstract base class with methods: `save(task)`, `get(task_id)`, `list(context_id, cursor, limit)`, `delete(task_id)`, `update_status(task_id, status)`.
2. `InMemoryTaskStore` default implementation stores tasks in a dictionary.
3. In-memory store supports TTL-based expiration (default: 1 hour, configurable).
4. In-memory store has configurable max capacity (default: 10,000 tasks).
5. `serve(registry, task_store=MyCustomStore())` accepts custom TaskStore implementations.
6. Task store operations are thread-safe.
7. Task store interface includes `save_push_config()` and `get_push_config()` methods for FR-012 integration.
8. Documentation provides a reference implementation guide for custom persistent stores.

**Dependencies:** None (foundational; soft dependency from FR-005, FR-008, FR-011, FR-012)

**Priority:** P1

---

#### FR-016: CLI Entry Point

**Title:** Command-line interface for launching A2A server without code

**Description:** Provide an `apcore-a2a` CLI command (and `python -m apcore_a2a`) that discovers modules from a specified directory and starts the A2A server. This enables quick testing and deployment without writing Python code.

```bash
# Basic usage
apcore-a2a serve --extensions-dir ./extensions

# With custom host, port, and name
apcore-a2a serve --extensions-dir ./extensions \
    --host 0.0.0.0 --port 9000 \
    --name "My Agent" --description "Production A2A agent"

# With JWT authentication
apcore-a2a serve --extensions-dir ./extensions \
    --auth-type bearer --auth-issuer https://idp.corp.com

# With push notifications enabled
apcore-a2a serve --extensions-dir ./extensions --push-notifications
```

**User Story:** US-010

**Acceptance Criteria:**
1. `apcore-a2a serve --extensions-dir ./extensions` starts an A2A server with discovered modules.
2. CLI options: `--host` (default: `0.0.0.0`), `--port` (default: `8000`), `--name`, `--description`, `--version`.
3. `--auth-type bearer --auth-issuer <url>` enables JWT authentication.
4. `--push-notifications` flag enables push notification support.
5. `--log-level` option accepts `debug`, `info`, `warning`, `error`.
6. `apcore-a2a --version` prints the package version and exits.
7. `apcore-a2a --help` prints usage information.
8. CLI is registered as a console entry point via `pyproject.toml` (`[project.scripts]`).
9. If `--extensions-dir` points to a non-existent directory, the CLI exits with a clear error and exit code 1.
10. Exit code 0 on clean shutdown, 1 on configuration error, 2 on runtime error.

**Dependencies:** FR-001

**Priority:** P1

---

### P2 -- Nice to Have (v2 Candidates)

---

#### FR-017: Agent Explorer UI (Browser-Based)

**Title:** Built-in browser UI for exploring skills, testing messages, and viewing tasks

**Description:** An optional, lightweight web UI served alongside the A2A HTTP server that allows developers to browse the agent's capabilities, compose test messages, view task state, and observe SSE streams in real time. This is the A2A counterpart to apcore-mcp's Explorer UI.

**Acceptance Criteria:**
1. When `explorer=True` is passed to `serve()`, the Explorer UI is mounted at `/explorer` (configurable prefix).
2. `GET /explorer/` returns an interactive HTML page displaying Agent Card metadata, Skills list, and test forms.
3. Explorer displays all Skills with descriptions, tags, examples, and input/output modes.
4. Explorer provides a form to compose and send `message/send` requests with result display.
5. Explorer supports `message/stream` with live SSE event rendering.
6. Explorer is a single self-contained HTML file (no external CDN dependencies).
7. Explorer is disabled by default; requires explicit `explorer=True` opt-in.
8. Explorer GET endpoints are exempt from JWT authentication; POST endpoints enforce auth when configured.

**Dependencies:** FR-001, FR-002, FR-004, FR-009

**Priority:** P2

---

#### FR-018: Health/Metrics Endpoints

**Title:** Operational health check and basic metrics for monitoring

**Description:** Provide health check and basic metrics endpoints for operational monitoring in production deployments.

**Acceptance Criteria:**
1. `GET /health` returns HTTP 200 with `{"status": "healthy", "module_count": N, "uptime_seconds": N}` when the server is operational.
2. `GET /metrics` returns JSON with: active tasks count, completed tasks count, failed tasks count, uptime, total request count.
3. Health endpoint checks task store connectivity.
4. Metrics endpoint is disabled by default; enabled via `serve(registry, metrics=True)`.
5. Metrics do not include any PII or task content.
6. Health endpoint does not require authentication.

**Dependencies:** FR-001, FR-015

**Priority:** P2

---

#### FR-019: Dynamic Skill Registration (Hot-Reload)

**Title:** Runtime module addition/removal with automatic Agent Card updates

**Description:** Support adding and removing modules from the Registry at runtime, with the Agent Card routing table automatically updating without server restart.

**Acceptance Criteria:**
1. When a module is added to the Registry via `registry.register()`, it appears in the Agent Card within 1 second.
2. When a module is removed via `registry.unregister()`, it is removed from the Agent Card within 1 second.
3. In-flight tasks for removed modules complete normally; new tasks targeting removed modules return -32601.
4. Hot-reload is triggered by Registry events (`registry.on("register", callback)`, `registry.on("unregister", callback)`).
5. The `skills[]` array in the Agent Card reflects the current state of the Registry at all times.
6. Agent Card changes are thread-safe.
7. An optional file-system watcher mode re-scans the extensions directory on changes.

**Dependencies:** FR-001, FR-002, FR-003

**Priority:** P2

---

**Feature Count Summary:**

| Priority | Count | Features |
|----------|-------|----------|
| P0       | 11    | FR-001 through FR-011 |
| P1       | 5     | FR-012 through FR-016 |
| P2       | 3     | FR-017 through FR-019 |
| **Total**| **19**|                        |

---

## 8. Non-Functional Requirements

### NFR-001: Performance

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Agent Card response time | < 10ms (p99) | Load test: 1,000 concurrent `GET /.well-known/agent.json` requests |
| `message/send` latency overhead | < 5ms above raw `Executor.call_async()` time | Benchmark: `message/send` vs direct Executor call on same module |
| SSE first event latency | < 50ms after request receipt | Benchmark: time from `message/stream` request to first `TaskStatusUpdateEvent` |
| Task store read latency | < 1ms for in-memory store | Benchmark: `tasks/get` under 10,000 stored tasks |
| Concurrent task support | >= 100 simultaneous active tasks | Load test: 100 parallel `message/send` requests |
| Memory overhead per task | < 10KB (excluding artifact content) | Memory profiling under sustained load |
| Agent Card generation time | < 100ms for 100 modules | Benchmark: Registry with 100 modules |
| `serve()` startup to first request | < 2 seconds | Integration test: time from `serve()` call to successful Agent Card fetch |

### NFR-002: Security

| Requirement | Detail |
|-------------|--------|
| No information leakage | ACL errors return generic messages; internal errors hide stack traces and file paths |
| Authentication support | JWT, Bearer, API Key; pluggable auth middleware via `Authenticator` protocol |
| Token validation | JWT signature verification, expiration checking, issuer and audience validation |
| CORS | Configurable allowed origins; default: none (same-origin only) |
| HTTPS | TLS termination supported via reverse proxy; webhook URLs must be HTTPS in production mode |
| Input sanitization | All client-provided strings sanitized before inclusion in logs or error messages |
| Dependency security | All dependencies must have no known critical CVEs at release time |

### NFR-003: Reliability

| Requirement | Detail |
|-------------|--------|
| Graceful shutdown | In-flight tasks complete (with configurable grace period, default: 30s) before server stops |
| Error isolation | A failing module does not crash the server or affect other in-flight tasks |
| State consistency | Task state machine prevents invalid transitions; concurrent access is thread-safe |
| Webhook resilience | Push notification failures do not block task execution; retries are asynchronous |
| Stream reconnection | `tasks/resubscribe` enables client reconnection to active task SSE streams |

### NFR-004: Compatibility

| Requirement | Detail |
|-------------|--------|
| A2A protocol version | Full compliance with A2A v0.3.0 specification |
| apcore-python version | Compatible with apcore-python >= 0.6.0 |
| a2a-sdk version | Uses official `a2a-sdk` Python package (latest stable) |
| Python version | Python >= 3.10 |
| ASGI compatibility | The ASGI app (from `async_serve()`) can be mounted in any ASGI server (uvicorn, hypercorn, daphne) |
| Interoperability | Verified against A2A reference client; compatible with non-apcore A2A agents |
| JSON-RPC compliance | Fully compliant with JSON-RPC 2.0 specification |

### NFR-005: Maintainability

| Requirement | Detail |
|-------------|--------|
| Test coverage | >= 90% line coverage; >= 85% branch coverage |
| Test count | Target ~400-500 tests (comparable to apcore-mcp-python's ~450) |
| Code documentation | All public APIs have docstrings with usage examples |
| Type annotations | 100% type-annotated public API; mypy strict mode passes |
| Architecture | Mirror apcore-mcp-python's layered architecture: `adapters/`, `server/`, `auth/`, `store/`, `client/` |
| Dependency count | Minimize direct dependencies: apcore-python, a2a-sdk, starlette, uvicorn, httpx, PyJWT (optional) |
| Package size | Core logic <= 1,500 lines (excluding tests, docs, and Explorer UI assets) |

---

## 9. Success Metrics & KPIs

### 9.1 Launch Metrics (At Release)

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Schema mapping accuracy | 100% of apcore module fields correctly mapped to A2A Skill and Agent Card fields | Automated test suite with full field coverage |
| Integration time | < 5 minutes from `pip install` to working A2A server | Timed walkthrough with documented quickstart |
| Lines of user code for basic A2A server | <= 5 lines | Count lines in quickstart example |
| A2A method coverage | 100% of v0.3.0 methods implemented | Feature checklist: message/send, message/stream, tasks/get, tasks/list, tasks/cancel, tasks/resubscribe, push notification CRUD, agentCard/get |
| Error mapping coverage | 100% of apcore error types mapped | Unit tests for each error type |
| Test coverage | >= 90% line coverage | `pytest --cov` report |
| P0 feature completion | 11/11 features pass acceptance criteria | Feature acceptance testing |
| Package size | Core logic <= 1,500 lines | `cloc src/apcore_a2a/ --exclude-dir=tests` |

### 9.2 Adoption Metrics (Post-Launch)

| Metric | Target (6 months) | Target (12 months) | Measurement Method |
|--------|-------------------|---------------------|-------------------|
| PyPI downloads | 1,000/month | 5,000/month | PyPI stats API |
| GitHub stars | 100 | 500 | GitHub API |
| Active deployments (estimated) | 50 | 200 | Opt-in telemetry + community survey |
| xxx-apcore projects with A2A support | 2 | 5 | Manual tracking of downstream adoption |

### 9.3 Quality Metrics (Ongoing)

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| A2A conformance test pass rate | 100% | A2A conformance test suite (when available) |
| P0/P1 bug fix time | < 48 hours | Issue tracker SLA monitoring |
| Documentation completeness | All public APIs documented with examples | Automated doc coverage check |
| Interoperability verified with N implementations | >= 3 (non-apcore A2A agents) | Manual testing + CI matrix |

### 9.4 Ecosystem Metrics (12 months)

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Community-contributed task store backends | >= 2 (e.g., Redis, PostgreSQL) | GitHub contributions |
| Enterprise pilot deployments | >= 3 | Partner engagement tracking |
| apcore dual-protocol demos (MCP + A2A) | >= 5 published examples | Documentation and blog tracking |

### 9.5 Qualitative Success Indicators

- **Developer sentiment:** Early adopters describe setup as "trivially easy" or "it just works."
- **Ecosystem validation:** At least one xxx-apcore project (comfyui-apcore) adopts apcore-a2a instead of building its own A2A server.
- **Technical credibility:** The dual-protocol capability (MCP + A2A from single module definition) is cited as a differentiator in framework comparisons.
- **Community contribution:** At least one external contributor submits a PR or issue within 2 months of launch.

---

## 10. Dependencies & Risks

### 10.1 Technical Dependencies

| Dependency | Version Constraint | Purpose | Risk Level |
|------------|-------------------|---------|------------|
| **apcore-python** | >= 0.6.0 | Registry, Executor, Module, Schema APIs | Low -- stable, under same organization's control |
| **a2a-sdk** | Latest stable | A2A protocol types, JSON-RPC handling, SSE utilities | Medium -- external project; pre-1.0 API may change |
| **starlette** | >= 0.27 | ASGI framework for HTTP server, SSE response handling | Low -- mature, widely deployed |
| **uvicorn** | >= 0.23 | ASGI server for development and production | Low -- mature, standard choice for Python async servers |
| **httpx** | >= 0.24 | HTTP client for A2A Client and push notification delivery | Low -- mature, async-native |
| **PyJWT** | >= 2.8 (optional) | JWT token validation for authentication middleware | Low -- mature, well-maintained |
| Python | >= 3.10 | Modern type hints, `match` statements, async features | Low -- aligns with apcore-python minimum |

### 10.2 Risks

| Risk | Likelihood | Impact | Severity | Mitigation |
|------|:----------:|:------:|:--------:|------------|
| **A2A protocol breaking changes before 1.0** | Medium | High | High | Pin to a2a-sdk compatible release; abstract A2A types behind internal adapter interfaces; monitor A2A spec repo for breaking change announcements |
| **a2a-sdk API instability** | Medium | Medium | Medium | Wrap a2a-sdk usage in thin adapter layer; write integration tests against SDK; maintain compatibility test matrix across SDK versions |
| **apcore-python API changes before 1.0** | Low | Medium | Low | Duck-typing throughout (no hard type checks on apcore types); integration tests against multiple apcore-python versions |
| **Skill targeting ambiguity** | Medium | Medium | Medium | Require explicit `skillId` in message metadata as primary routing; support configurable default skill as fallback; defer NLP-based content routing |
| **Task store memory pressure under load** | Medium | Low | Low | Default TTL expiration (1 hour); configurable capacity limits (10,000 tasks); clear documentation on when to upgrade to persistent stores |
| **SSE stream disconnection handling** | Low | Medium | Low | Implement `tasks/resubscribe` for reconnection; configurable task cancellation on disconnect; heartbeat events to detect stale connections |
| **Security scheme complexity** | Medium | Medium | Medium | Start with JWT/Bearer (most common); defer OAuth 2.0 flow to reverse proxy; document recommended enterprise deployment patterns |
| **Multi-turn context memory growth** | Low | Low | Low | Configurable context history limit (default: 100 messages); TTL on context data; documentation on sizing and capacity planning |
| **Enterprise firewall blocking SSE** | Low | Medium | Low | Push notifications as alternative to SSE; documentation on proxy configuration for SSE passthrough |
| **Low initial adoption due to small ecosystem** | Medium | Medium | Medium | Ship compelling demo with mock modules; integrate with comfyui-apcore early; dual-protocol demo (MCP + A2A) as showcase |

---

## 11. "What If We Don't Build This?" Analysis

### Immediate Consequences

1. **apcore modules remain MCP-only.** Modules can serve AI coding assistants (Claude Desktop, Cursor) but cannot participate in agent-to-agent collaboration workflows. This is an increasingly significant limitation as multi-agent architectures become the dominant pattern for complex AI workflows in 2025-2026.

2. **Manual implementation burden persists.** Developers who need A2A must build the Agent Card, JSON-RPC handling, task lifecycle state machine, SSE streaming, and error mapping from scratch. Based on A2A specification complexity, this is realistically 500+ lines of boilerplate and 2-4 weeks of engineering effort per project.

3. **Schema duplication becomes unavoidable.** Without automatic mapping, developers must manually re-describe in A2A format the same capabilities already defined in apcore module metadata. This creates maintenance burden and inevitable schema drift.

### Strategic Consequences

4. **Ecosystem positioning weakens significantly.** LangChain and CrewAI are shipping A2A integrations within their ecosystems. If apcore lacks A2A support, developers evaluating agent frameworks will see apcore as "MCP-only" while competitors offer dual-protocol support. The "universal framework" value proposition becomes difficult to defend.

5. **xxx-apcore projects are half-connected.** Every domain-specific apcore bridge (comfyui-apcore, vnpy-apcore, blender-apcore, etc.) would be limited to MCP. This halves the value proposition for these projects and their users who increasingly need agent-to-agent interoperability.

6. **Enterprise adoption barrier.** Enterprise multi-agent deployments increasingly mandate A2A compliance (driven by Google, Linux Foundation, and the 50+ partner ecosystem). Without apcore-a2a, apcore-powered agents cannot participate in these environments, closing off a major growth vector.

### Honest Assessment

Not building apcore-a2a does not kill the apcore project. apcore-mcp provides meaningful value for the MCP use case, and apcore's schema-driven module framework remains useful independently. However, not building it means:

- apcore is positioned as a "module framework for AI assistants" rather than a "universal AI interoperability framework."
- The addressable market is roughly halved (MCP-only vs. MCP + A2A).
- The competitive moat is weaker: any framework can do one protocol; few can do both from a single source.
- The network effect from xxx-apcore projects is diminished since each project only gains one protocol surface.

The investment (estimated 800-1,500 lines of core logic, following the proven apcore-mcp adapter pattern) is modest relative to the strategic value of completing the protocol surface area and establishing apcore as the only schema-driven framework with native dual-protocol coverage.

---

## 12. Timeline & Milestones

### Phase 1: Foundation (Weeks 1-3)

| Milestone | Features | Deliverable |
|-----------|----------|-------------|
| M1.1: Adapters | FR-003 (Skill mapping), FR-007 (Error mapping) | Schema adapters, error mappers -- the foundational conversion layer |
| M1.2: Agent Card & Server | FR-001 (serve()), FR-002 (Agent Card), FR-015 (Task store) | Working server with Agent Card endpoint and in-memory task storage |
| M1.3: Synchronous Execution | FR-004 (message/send), FR-005 (Task lifecycle), FR-006 (Execution routing) | Complete synchronous request-response path through Executor pipeline |

**Phase 1 Exit Criteria:** `serve(registry)` starts a server; `GET /.well-known/agent.json` returns a valid Agent Card with all modules as Skills; `message/send` executes a module and returns a completed Task with correct Artifacts.

### Phase 2: Streaming & Tasks (Weeks 4-5)

| Milestone | Features | Deliverable |
|-----------|----------|-------------|
| M2.1: Task Management | FR-008 (tasks/get, list, cancel) | Task query, listing, and cancellation endpoints |
| M2.2: SSE Streaming | FR-009 (message/stream) | Complete streaming path with TaskStatusUpdateEvent and TaskArtifactUpdateEvent |
| M2.3: Multi-Turn | FR-011 (contextId conversations) | Multi-turn conversation support with input elicitation |

**Phase 2 Exit Criteria:** All three interaction modes work (synchronous, streaming, multi-turn). Task management endpoints are operational. Multi-turn conversations with `input_required` state function correctly.

### Phase 3: Client & Extensions (Weeks 6-7)

| Milestone | Features | Deliverable |
|-----------|----------|-------------|
| M3.1: A2A Client | FR-010 (A2AClient) | Standalone client for discovering and invoking remote A2A agents |
| M3.2: Push Notifications | FR-012 (Webhook delivery) | Complete push notification support with retry logic |
| M3.3: Authentication | FR-013 (JWT bridging), FR-014 (Extended Agent Card) | Auth middleware, identity bridging, and extended Agent Card |

**Phase 3 Exit Criteria:** Client can discover and invoke remote A2A agents. Push notifications work end-to-end. JWT authentication bridges to apcore Identity. Extended Agent Card serves privileged Skills.

### Phase 4: Polish & Release (Weeks 8-9)

| Milestone | Features | Deliverable |
|-----------|----------|-------------|
| M4.1: CLI | FR-016 (CLI entry point) | `apcore-a2a` command with full option support |
| M4.2: Explorer & Operations | FR-017 (Explorer UI), FR-018 (Health/Metrics) | Browser-based exploration and operational endpoints |
| M4.3: Dynamic Registration | FR-019 (Hot-reload) | Runtime module addition/removal |
| M4.4: Release Preparation | All features | Documentation, packaging, PyPI publication |

**Phase 4 Exit Criteria:** >= 90% test coverage. All P0 and P1 features complete and passing acceptance criteria. Documentation published (quickstart, API reference, deployment guide). Package published to PyPI as `apcore-a2a`.

### Post-Release

- Interoperability testing with 3+ non-apcore A2A implementations.
- Community feedback collection (4-week window).
- v1.1.0 planning based on feedback and A2A spec evolution.
- Dual-protocol demo showcasing MCP + A2A from the same apcore Registry.

### Launch Criteria

The following must all be true before the v1.0.0 release:

1. All P0 features (FR-001 through FR-011) pass their acceptance criteria.
2. All P1 features (FR-012 through FR-016) pass their acceptance criteria.
3. Test coverage >= 90% on core logic (`src/apcore_a2a/`).
4. Verified working with at least one A2A reference client.
5. Verified working with at least one non-apcore A2A agent (interoperability).
6. Agent Card validates against A2A v0.3.0 JSON Schema.
7. Quickstart documentation exists and is verified by a developer who did not write the code.
8. `pyproject.toml` is complete with correct dependencies, metadata, and entry points.
9. No known P0 or P1 bugs.

### Documentation Requirements

| Document | Purpose | Required for Launch? |
|----------|---------|:-------------------:|
| README.md | Project overview, quickstart, installation | Yes |
| Quickstart Guide | Step-by-step: install, discover, serve, verify | Yes |
| API Reference | `serve()`, `async_serve()`, `A2AClient` signatures and parameters | Yes |
| Deployment Guide | Production deployment patterns (reverse proxy, TLS, auth) | Yes |
| Schema Mapping Reference | Complete apcore-to-A2A mapping tables | Yes |
| Architecture Overview | Internal design for contributors | No (post-launch) |
| Migration Guide | N/A for v1; needed for future breaking changes | No |

---

## 13. Open Questions

| ID | Question | Impact | Status | Proposed Resolution |
|----|----------|--------|--------|---------------------|
| OQ-001 | How should structured JSON input be conveyed via A2A's text-based Message Parts? | FR-004, FR-006 | Open | Support both: (1) DataPart with `application/json` media type for structured input, (2) TextPart with JSON string that is auto-parsed based on Skill's declared `inputModes`. Prioritize DataPart for programmatic callers. |
| OQ-002 | Should the Agent Card expose `output_schema` information? A2A has no native field for this. | FR-002, FR-003 | Open | Use Skill `extensions` field (A2A v0.3.0 supports extensions) to include `outputSchema` as custom metadata. This is non-breaking and discoverable by apcore-aware clients. Non-apcore clients ignore extensions. |
| OQ-003 | How should apcore's `requires_approval` map to A2A's `input_required` state? | FR-005, FR-006, FR-011 | Open | Map to `input_required` state with a structured message indicating approval is needed (type: "approval_request", module_id, description, arguments). Follow-up message with approval confirmation resumes execution. This is semantically closest to A2A's intent for human-in-the-loop scenarios. |
| OQ-004 | How should `contextId` persistence work across server restarts? | FR-011, FR-015 | Open | Delegate to task store implementation. In-memory store loses context on restart (documented limitation). Persistent stores (Redis, PostgreSQL) maintain context across restarts. Document this as a deployment consideration with guidance on when to use persistent stores. |
| OQ-005 | Should the A2A Client be importable independently without server dependencies? | FR-010 | Open | Yes. Structure the package so `from apcore_a2a.client import A2AClient` does not import server-side dependencies (starlette, uvicorn). Use lazy imports or optional dependency groups (`pip install apcore-a2a[client]`). |
| OQ-006 | How to handle skill targeting when a message does not specify a target module? | FR-004, FR-006 | Open | Three strategies ordered by priority: (1) Require explicit `skillId` in message metadata (primary), (2) Default skill configuration via `serve(registry, default_skill="module.id")`, (3) Content-based routing using keyword matching to skill descriptions (deferred). Start with (1) and (2) in v1. |
| OQ-007 | What is the versioning strategy when A2A protocol releases breaking changes? | All features | Open | Support the latest A2A spec version at time of release. Maintain a `PROTOCOL_VERSION` constant. When A2A releases breaking changes, cut a new major version of apcore-a2a. Maintain compatibility matrix in documentation. |

---

## 14. Appendix

### A. Key Schema Mappings

#### A.1 apcore Registry --> A2A Agent Card

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

#### A.2 apcore Module --> A2A Skill

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
(custom extension)              -->   Skill.extensions.inputSchema
(custom extension)              -->   Skill.extensions.outputSchema
```

#### A.3 apcore Annotations --> A2A Capabilities

```
apcore Annotations                    A2A Agent Card
========================              ========================
any module has streaming=true   -->   capabilities.streaming = true
push_notifications enabled      -->   capabilities.pushNotifications = true
task store supports history     -->   capabilities.stateTransitionHistory = true
readonly annotation             -->   Skill.extensions.annotations.readonly
destructive annotation          -->   Skill.extensions.annotations.destructive
idempotent annotation           -->   Skill.extensions.annotations.idempotent
requires_approval annotation    -->   input_required state behavior
open_world annotation           -->   Skill.extensions.annotations.open_world
```

#### A.4 apcore Execution --> A2A Task Lifecycle

```
apcore Executor                       A2A Task
========================              ========================
executor.call_async() start     -->   status.state = "submitted"
module.execute() running        -->   status.state = "working"
executor.call_async() success   -->   status.state = "completed"
executor.call_async() error     -->   status.state = "failed"
ApprovalPendingError            -->   status.state = "input_required"
CancelToken.cancel()            -->   status.state = "canceled"
ACLDeniedError                  -->   status.state = "failed" (generic message)
executor.stream() chunk         -->   TaskArtifactUpdateEvent (append: true)
executor.stream() complete      -->   TaskStatusUpdateEvent (completed)
```

#### A.5 apcore Errors --> A2A JSON-RPC Errors

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

### B. Architecture Diagram

```
                        apcore-a2a Package
 +--------------------------------------------------------------+
 |                                                              |
 |  Public API                                                  |
 |  +----------------------------------------------------------+|
 |  |  serve(registry, **options)   async_serve(registry)      ||
 |  |  A2AClient(url)                                          ||
 |  +-------------------------+--------------------------------+|
 |                            |                                 |
 |  Server Layer              |                                 |
 |  +-------------------------v--------------------------------+|
 |  |  server/                                                 ||
 |  |  +-- factory.py      (server construction)               ||
 |  |  +-- router.py       (JSON-RPC method dispatch)          ||
 |  |  +-- transport.py    (HTTP + SSE handling)               ||
 |  |  +-- task_manager.py (task lifecycle state machine)      ||
 |  |  +-- push.py         (webhook notification delivery)     ||
 |  +-------------------------+--------------------------------+|
 |                            |                                 |
 |  Adapter Layer             |                                 |
 |  +-------------------------v--------------------------------+|
 |  |  adapters/                                               ||
 |  |  +-- schema.py       (module --> Skill conversion)       ||
 |  |  +-- agent_card.py   (Registry --> Agent Card)           ||
 |  |  +-- errors.py       (apcore error --> A2A error)        ||
 |  |  +-- execution.py    (A2A message --> Executor call)     ||
 |  |  +-- annotations.py  (annotations --> capabilities)      ||
 |  +-------------------------+--------------------------------+|
 |                            |                                 |
 |  Auth Layer                |      Storage Layer              |
 |  +-------------------------+--+ +---------------------------+|
 |  |  auth/                     | |  store/                   ||
 |  |  +-- middleware.py         | |  +-- interface.py (ABC)   ||
 |  |  +-- jwt.py                | |  +-- memory.py (default)  ||
 |  |  +-- identity_bridge.py   | +---------------------------+|
 |  +----------------------------+                              |
 |                                                              |
 |  Client Layer                                                |
 |  +----------------------------------------------------------+|
 |  |  client/                                                 ||
 |  |  +-- client.py       (A2AClient class)                   ||
 |  |  +-- discovery.py    (Agent Card fetching + caching)     ||
 |  |  +-- errors.py       (typed A2A error exceptions)        ||
 |  +----------------------------------------------------------+|
 |                                                              |
 +--------------------------------------------------------------+
            |                               |
            v                               v
   +-----------------+            +------------------+
   |  apcore-python   |            |  a2a-sdk          |
   |  (Registry,      |            |  (A2A types,      |
   |   Executor,      |            |   JSON-RPC,       |
   |   Module,        |            |   SSE utilities)   |
   |   Schema)        |            |                    |
   +-----------------+            +------------------+
```

### C. apcore API Surface Used by apcore-a2a

For reference, the following apcore-python APIs are consumed by apcore-a2a:

```python
# Registry (module discovery and query)
registry = Registry(extensions_dir="./extensions")
registry.discover() -> int
registry.list(tags=None, prefix=None) -> list[str]
registry.get(module_id) -> module | None
registry.get_definition(module_id) -> ModuleDescriptor | None
registry.iter() -> Iterator[tuple[str, Any]]
registry.on("register", callback)
registry.on("unregister", callback)

# Executor (execution pipeline)
executor = Executor(registry, middlewares=None, acl=None, config=None)
executor.call_async(module_id, inputs, context=None) -> dict  # async
executor.stream(module_id, inputs, context=None) -> AsyncIterator  # async
executor.validate(module_id, inputs) -> ValidationResult

# ModuleDescriptor (schema data)
descriptor.module_id: str
descriptor.description: str
descriptor.input_schema: dict[str, Any]   # JSON Schema
descriptor.output_schema: dict[str, Any]  # JSON Schema
descriptor.annotations: ModuleAnnotations | None
descriptor.name: str | None
descriptor.documentation: str | None
descriptor.tags: list[str]
descriptor.version: str
descriptor.examples: list[ModuleExample]

# ModuleAnnotations (behavioral flags)
annotations.readonly: bool
annotations.destructive: bool
annotations.idempotent: bool
annotations.requires_approval: bool
annotations.open_world: bool

# Error hierarchy (all extend ModuleError)
ModuleNotFoundError(module_id)
SchemaValidationError(message, errors)
ACLDeniedError(caller_id, target_id)
ModuleTimeoutError(module_id, timeout_ms)
InvalidInputError(message)
CallDepthExceededError(depth, max_depth, call_chain)
CircularCallError(module_id, call_chain)
CallFrequencyExceededError(module_id, count, max_repeat, call_chain)
ApprovalPendingError(module_id)
```

### D. Feature Traceability Matrix

| Feature ID | User Story | Phase | Priority | Depends On |
|------------|------------|-------|----------|------------|
| FR-001 | US-001 | 1 | P0 | FR-002, FR-003, FR-004, FR-005, FR-006, FR-007 |
| FR-002 | US-002 | 1 | P0 | FR-003 |
| FR-003 | US-002 | 1 | P0 | -- |
| FR-004 | US-003 | 1 | P0 | FR-003, FR-005, FR-006, FR-007 |
| FR-005 | US-005 | 1 | P0 | FR-015 (soft) |
| FR-006 | US-003 | 1 | P0 | FR-005, FR-007 |
| FR-007 | US-003 | 1 | P0 | -- |
| FR-008 | US-005 | 2 | P0 | FR-005, FR-015 |
| FR-009 | US-004 | 2 | P0 | FR-004, FR-005, FR-006 |
| FR-010 | US-007 | 3 | P0 | -- |
| FR-011 | US-006 | 2 | P0 | FR-004, FR-005, FR-015 |
| FR-012 | US-008 | 3 | P1 | FR-005, FR-008 |
| FR-013 | US-009 | 3 | P1 | FR-001, FR-006 |
| FR-014 | US-009 | 3 | P1 | FR-002, FR-013 |
| FR-015 | US-005 | 1 | P1 | -- |
| FR-016 | US-010 | 4 | P1 | FR-001 |
| FR-017 | -- | 4 | P2 | FR-001, FR-002, FR-004, FR-009 |
| FR-018 | -- | 4 | P2 | FR-001, FR-015 |
| FR-019 | -- | 4 | P2 | FR-001, FR-002, FR-003 |

### E. Glossary

| Term | Definition |
|------|-----------|
| **A2A** | Agent-to-Agent protocol. Open standard by Google/Linux Foundation for inter-agent communication. Enables diverse AI agents to discover, communicate, and collaborate on tasks. |
| **Agent Card** | JSON document describing an A2A agent's identity, capabilities, skills, and security schemes. Served at `/.well-known/agent.json` for standard discovery. |
| **apcore** | Schema-driven module development framework. Protocol-agnostic core with adapter-based protocol support (MCP via apcore-mcp, A2A via apcore-a2a). |
| **Artifact** | Output produced by an A2A task. Contains Parts (TextPart, DataPart, FilePart) representing the result of module execution. |
| **contextId** | Unique identifier (UUID v4) grouping multiple messages into a multi-turn conversation. Enables stateful interactions and input elicitation. |
| **Executor** | apcore component that executes modules through the full pipeline: context creation, safety checks, ACL enforcement, input validation, middleware, execution, output validation. |
| **JSON-RPC 2.0** | Lightweight remote procedure call protocol using JSON encoding. Primary protocol binding for A2A communication. |
| **MCP** | Model Context Protocol. Anthropic's protocol for model-to-tool communication. Complementary to A2A: MCP connects AI assistants to tools; A2A connects agents to agents. |
| **Module** | An apcore capability unit with defined `input_schema`, `output_schema`, `description`, `annotations`, `tags`, and `examples`. The atomic unit of functionality in the apcore ecosystem. |
| **Part** | A piece of content within an A2A Message or Artifact. Types: TextPart (plain text), DataPart (structured data with media type), FilePart (file reference). |
| **Push Notification** | Webhook-based delivery of task state updates via HTTP POST. Third A2A interaction mode alongside synchronous and streaming. |
| **Registry** | apcore component that discovers, loads, and indexes modules. Provides `list()`, `get_definition()`, and event-based registration notifications. |
| **Skill** | An A2A concept representing a specific capability of an agent. Maps 1:1 to apcore modules in apcore-a2a. |
| **SSE** | Server-Sent Events. HTTP-based unidirectional streaming mechanism used for A2A `message/stream` to deliver `TaskStatusUpdateEvent` and `TaskArtifactUpdateEvent`. |
| **Task** | Stateful A2A work unit with lifecycle: submitted --> working --> completed/failed/canceled/input_required. Core concept for managing asynchronous and long-running operations. |
| **Task Store** | Pluggable storage backend for persisting task state. In-memory default; can be replaced with Redis, PostgreSQL, or any custom implementation conforming to the `TaskStore` interface. |
| **xxx-apcore** | Convention for apcore adapter projects targeting specific domains: `comfyui-apcore`, `vnpy-apcore`, `blender-apcore`, etc. Each gains A2A capability by adding apcore-a2a as a dependency. |

### F. References

| Reference | Description |
|-----------|-------------|
| `ideas/apcore-a2a.md` | Original idea document with validation status, schema mappings, and competitive analysis |
| apcore-python SDK | Python SDK providing Registry, Executor, ModuleDescriptor, ModuleAnnotations, error hierarchy |
| [A2A Protocol Specification (v0.3.0)](https://google.github.io/A2A/) | Official Agent-to-Agent protocol specification by Google/Linux Foundation |
| [A2A Python SDK (a2a-sdk)](https://pypi.org/project/a2a-sdk/) | Official Python SDK for building A2A agents and clients |
| apcore-mcp PRD | Reference PRD for the sibling MCP adapter -- architectural pattern reference |
| apcore SCOPE.md | apcore's scope document that designates MCP/A2A adapters as independent projects |

---

*End of Document*
