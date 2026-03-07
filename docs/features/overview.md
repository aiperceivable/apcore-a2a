# apcore-a2a Feature Specs Overview

This directory contains implementation-level feature specifications for **apcore-a2a** — the automatic A2A protocol adapter for the apcore module registry.

## Feature Map

| Feature | Priority | Depends On | Description |
|---------|----------|------------|-------------|
| [adapters](adapters.md) | P0 | None (foundation) | Pure-logic converters: AgentCardBuilder, SkillMapper, MessageAdapter, ResultMapper, ErrorMapper |
| [storage](storage.md) | P0 | None | Pluggable task persistence — TaskStore protocol + InMemoryTaskStore |
| [server-core](server-core.md) | P0 | adapters, storage | Core server: A2AServerFactory, ExecutionRouter, TaskManager, TransportManager |
| [streaming](streaming.md) | P0 | server-core | SSE streaming for `message/stream` and `tasks/resubscribe` |
| [push-notifications](push-notifications.md) | P1 | storage, server-core | Webhook-based async delivery of task state changes |
| [auth](auth.md) | P1 | None (standalone) | JWT/Bearer authentication middleware |
| [client](client.md) | P0 | None (standalone) | HTTP client: A2AClient + AgentCardFetcher |
| [public-api](public-api.md) | P0 | All above | Top-level entry: `serve()`, `async_serve()`, re-exports |
| [cli](cli.md) | P1 | public-api | CLI entry point for launching A2A servers |
| [explorer](explorer.md) | P2 | server-core | Browser-based interactive UI for testing agents |
| [ops](ops.md) | P2 | server-core | Health, metrics, and dynamic module registration |

## Dependency Graph

```
adapters ─┐
           ├─→ server-core ─┬─→ streaming ──────┐
storage ──┘                 ├─→ push-notifications│
                            ├─→ explorer          ├─→ public-api ─→ cli
                            ├─→ ops               │
auth ───────────────────────┘                     │
client ───────────────────────────────────────────┘
```

## Related Documents

- [Getting Started](../getting-started.md) — Installation & usage guide (Python & TypeScript)
- [PRD](../prd.md) — Product Requirements Document
- [SRS](../srs.md) — Software Requirements Specification
- [Tech Design](../tech-design.md) — Technical Design Document
- [Test Plan](../test-plan.md) — Test Plan
