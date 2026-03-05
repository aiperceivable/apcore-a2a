# Idea: apcore-a2a

> apcore Module Registry to A2A (Agent-to-Agent) Protocol automatic bridge

## Status: Validated

## Problem Statement

apcore modules are inherently AI-perceivable (with `input_schema`, `output_schema`, `description`, `annotations`), and apcore-mcp already bridges them to MCP clients (Claude, Cursor). However, apcore modules **cannot participate in agent-to-agent collaboration**. The A2A (Agent-to-Agent) protocol — an open standard by Google/Linux Foundation — enables diverse AI agents to discover each other, communicate, and collaborate on tasks. Manually implementing an A2A server for apcore modules requires significant effort: Agent Card construction, JSON-RPC endpoint handling, task lifecycle management, SSE streaming, and more.

## Proposed Solution

An independent adapter package that automatically converts any apcore Module Registry into a fully functional **A2A-compliant agent server**. Modules are exposed as A2A Skills, execution is routed through apcore's Executor pipeline, and the full A2A task lifecycle (submitted → working → completed) is managed automatically.

```python
from apcore import Registry
from apcore_a2a import serve

registry = Registry(extensions_dir="./extensions")
registry.discover()

# A2A Server mode — expose all modules as A2A agent skills
serve(registry)

# Agent Card automatically available at:
# GET /.well-known/agent.json
```

## Core Value Proposition

- **Zero manual Agent Card authoring**: apcore module metadata auto-maps to A2A Agent Card, Skills, and capabilities
- **Full task lifecycle**: A2A Task states (submitted → working → input_required → completed/failed) managed automatically
- **Streaming support**: apcore streaming modules map to A2A SSE (Server-Sent Events) with `TaskStatusUpdateEvent` and `TaskArtifactUpdateEvent`
- **Multi-turn conversations**: A2A `contextId` enables stateful, multi-turn interactions across modules
- **Ecosystem multiplier**: every `xxx-apcore` project gains A2A agent capability via this one package
- **Complementary to apcore-mcp**: MCP = model-to-tool; A2A = agent-to-agent — same modules, two protocol surfaces
- **Enterprise-ready**: A2A security schemes map to apcore ACL + Identity system

## Key Schema Mapping

### apcore → A2A Agent Card

```
apcore Registry                     →  A2A Agent Card
─────────────────────────────────────────────────────
(config) project.name               →  AgentCard.name
(config) project.description        →  AgentCard.description
(config) project.version            →  AgentCard.version
(serve url)                         →  AgentCard.url
(computed from modules)             →  AgentCard.skills[]
(computed from annotations)         →  AgentCard.capabilities
```

### apcore Module → A2A Skill

```
apcore Module                       →  A2A Skill
─────────────────────────────────────────────────────
module_id                           →  Skill.id
description                         →  Skill.description
tags                                →  Skill.tags
examples[].title                    →  Skill.examples[]
input_schema (JSON Schema)          →  (inputModes: application/json)
output_schema (JSON Schema)         →  (outputModes: application/json)
```

### apcore Execution → A2A Task Lifecycle

```
apcore Executor                     →  A2A Task
─────────────────────────────────────────────────────
executor.call_async() start         →  Task.status.state = "submitted"
module.execute() running            →  Task.status.state = "working"
executor.call_async() success       →  Task.status.state = "completed"
executor.call_async() error         →  Task.status.state = "failed"
ApprovalPendingError                →  Task.status.state = "input_required"
CancelToken.cancel()                →  Task.status.state = "canceled"
ACLDeniedError                      →  Task.status.state = "rejected"
```

### apcore Annotations → A2A Capabilities

```
apcore Annotations                  →  A2A Agent Card
─────────────────────────────────────────────────────
any module has streaming=true       →  capabilities.streaming = true
(push notification config)          →  capabilities.pushNotifications = true
(state history config)              →  capabilities.stateTransitionHistory = true
```

### apcore Errors → A2A JSON-RPC Errors

```
apcore Error                        →  A2A Error Code
─────────────────────────────────────────────────────
ModuleNotFoundError                 →  -32601 (Method not found)
SchemaValidationError               →  -32602 (Invalid params)
ACLDeniedError                      →  -32001 (TaskNotFoundError, hide details)
ModuleExecuteError                  →  -32603 (Internal error)
ModuleTimeoutError                  →  -32603 (Internal error)
ApprovalPendingError                →  input_required state (not error)
```

## Architecture Position

Defined by apcore SCOPE.md:

- apcore core: "MCP/A2A Adapters → Won't Do → Independent adapter projects"
- apcore-a2a: independent adapter that uses the A2A SDK internally
- Does NOT reimplement A2A protocol; bridges apcore's registry to the existing A2A SDK
- Sibling project to apcore-mcp (same adapter pattern, different protocol)

```
apcore-python (core)
    ↓ provides Registry + Executor + Module + Schema
apcore-a2a (this project)
    ├── A2A Server Path:
    │     ↓ uses A2A Python SDK (a2a-sdk)
    │   A2A Agent Server (JSON-RPC / HTTP / gRPC)
    │     ↓
    │   Other A2A Agents / A2A Clients / Orchestrators
    │
    └── A2A Client Path:
          ↓ uses A2A Python SDK
        A2AClient — discover and call remote A2A agents
          ↓
        Remote A2A-compliant agents
```

## Scope

### In Scope

- **A2A Server (Core)**:
  - Agent Card generation from apcore Registry metadata
  - Agent Card serving at `/.well-known/agent.json`
  - Skill mapping: apcore modules → A2A Skills
  - JSON-RPC endpoint handling (`message/send`, `message/stream`, `tasks/get`, `tasks/cancel`, etc.)
  - Task lifecycle management (state machine: submitted → working → completed/failed/canceled)
  - Task storage (in-memory, pluggable interface for persistence)
  - Execution routing: A2A task → apcore Executor (with ACL, validation, middleware)
  - Error mapping: apcore errors → A2A JSON-RPC errors
  - SSE streaming: apcore streaming modules → A2A `TaskStatusUpdateEvent` + `TaskArtifactUpdateEvent`
  - Multi-turn conversations via `contextId`
  - Input elicitation: apcore approval → A2A `input_required` state

- **A2A Server (Extended)**:
  - Push notification support (webhook delivery)
  - Extended Agent Card (authenticated endpoint with additional skills)
  - Security scheme declaration in Agent Card
  - JWT/Bearer authentication bridging to apcore Identity

- **A2A Client (Bonus)**:
  - `A2AClient` class for discovering and calling remote A2A agents
  - Agent Card fetching and caching
  - `send_message()`, `stream_message()`, `get_task()`, `cancel_task()`

### Out of Scope

- A2A protocol implementation (use existing A2A SDK)
- Module definition (that's apcore-python's job)
- Domain-specific module wrapping (that's xxx-apcore projects' job)
- gRPC binding (JSON-RPC + HTTP REST first, gRPC later)
- Agent Card signing (future enhancement)
- Agent registry/directory service (external service)
- MCP adapter (that's apcore-mcp's job)

## Dependencies

- `apcore` (apcore-python SDK >= 0.6.0)
- `a2a-sdk` (official A2A Python SDK)
- `starlette` + `uvicorn` (HTTP server, may be bundled with a2a-sdk)
- `PyJWT` (optional, for JWT authentication)

## Target Users

1. **apcore module developers** who want their modules discoverable and callable by other AI agents
2. **xxx-apcore project developers** (comfyui-apcore, vnpy-apcore, etc.) who want A2A agent capability
3. **AI agent orchestrators** who need to discover and coordinate with apcore-powered agents
4. **Enterprise teams** building multi-agent systems where agents need interoperability

## Competitive Analysis

| Approach | How it works | Limitation |
|----------|-------------|-----------|
| Manual A2A server | Hand-write Agent Card, task lifecycle, JSON-RPC | Tedious, 500+ lines boilerplate |
| A2A SDK directly | Build agent from scratch using SDK | No schema reuse, no standardization |
| LangChain A2A | Framework-specific integration | Locked to LangChain ecosystem |
| **apcore-a2a** | Auto-bridge from apcore Registry | Requires apcore modules (by design) |

## Demand Validation

- A2A protocol adopted by 50+ partners (Google, Atlassian, Salesforce, SAP, ServiceNow, LangChain)
- A2A moved to Linux Foundation governance (April 2025), signaling long-term stability
- Multi-agent systems are the dominant architecture pattern for complex AI workflows (2025-2026)
- Enterprise demand for agent interoperability is the #1 cited need in AI platform surveys
- apcore already has the exact metadata A2A needs (schema, description, annotations, tags, examples)
- apcore-mcp proves the adapter pattern works — A2A is the natural next step
- The mapping from apcore metadata to A2A concepts is clean and nearly 1:1

## What If We Don't Build This?

- apcore modules remain isolated within MCP ecosystem — cannot participate in agent-to-agent collaboration
- Users who need A2A must manually build server infrastructure (Agent Card, task lifecycle, streaming)
- Competing frameworks (LangChain, CrewAI) gain A2A support first, capturing the multi-agent market
- The apcore ecosystem misses the opportunity to prove its "universal framework" value proposition
- xxx-apcore projects can only serve MCP clients, not peer agents

## Development Estimate

- **Scope**: ~800-1500 lines of core logic + tests + docs
- **Architecture**: Mirror apcore-mcp-python structure (adapters, server, auth)
- **Priority**: Build AFTER apcore-mcp stabilization, as it's the second protocol surface

## Success Criteria

1. Any apcore Registry can be served as A2A agent with one function call
2. Agent Card automatically generated and served at `/.well-known/agent.json`
3. All modules correctly mapped to A2A Skills
4. Full task lifecycle (submitted → working → completed/failed/canceled) works
5. SSE streaming works for streaming-capable modules
6. Multi-turn conversations work via contextId
7. Execution goes through full apcore pipeline (ACL, validation, middleware)
8. Compatible with official A2A SDK clients and other A2A agents

## Relationship to Other Projects

- **apcore** (protocol spec): defines the standard this project implements against
- **apcore-python**: provides the Registry/Executor API this project consumes
- **apcore-mcp / apcore-mcp-python**: sibling adapter — same pattern, different protocol (MCP)
- **comfyui-apcore**: consumer — ComfyUI nodes → apcore → A2A agent
- **A2A SDK (a2a-sdk)**: upstream dependency — official A2A protocol implementation

## Open Questions

1. How to handle apcore modules that need structured JSON input via A2A's text-based Message Parts?
2. Should the Agent Card expose `output_schema` information (A2A doesn't have a native field for this)?
3. How to map apcore's `requires_approval` to A2A's `input_required` vs `auth_required` states?
4. How to handle `contextId` persistence across server restarts?

## Resolved Questions

1. ~~Should A2A Client functionality be in-scope for v1, or deferred to v2?~~ **v1 includes both Server and Client** — apcore agents should both be callable and able to call other agents.
2. ~~Should we support A2A push notifications in v1, or defer?~~ **v1 includes Push Notifications** — complete coverage of all three A2A interaction modes (sync, streaming, push).
3. ~~Which A2A SDK to use?~~ **Official `a2a-sdk` (Python)** — Google/Linux Foundation official SDK, `pip install a2a-sdk`.
