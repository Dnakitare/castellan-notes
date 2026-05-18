# Architecture

Castellan is a small set of containerized services that together demonstrate tiered local LLM use under a delegated-authority model. The architecture is intentionally boring in its choice of protocols (HTTP, MCP, NATS, JSON) and intentionally specific in where authority lives.

## Components

Each component is a separate container. The seams are chosen so that any component can be swapped without touching the others, and so that the authority boundaries are physical, not just logical.

### `small-inference`
- **Purpose**: Serves the small (3B-class) classifier model.
- **Interface**: OpenAI-compatible HTTP `/v1/chat/completions`.
- **Implementation**: `ghcr.io/ggml-org/llama.cpp:server` with the GGUF mounted from a named volume populated by a one-shot init container (`small-inference-init`) that fetches the model via curl. Swappable for any OpenAI-compatible server. See `docs/decisions/0001-inference-runtime.md`.
- **State**: Stateless. Long-lived. GPU-attached if available, CPU-only fallback for development.

### `mid-inference`
- **Purpose**: Serves the mid-tier (8B–14B-class) reasoning model.
- **Interface**: OpenAI-compatible HTTP `/v1/chat/completions`.
- **Implementation**: Same as `small-inference`, with its own model volume and `mid-inference-init` container.
- **State**: Stateless. Long-lived.

### `rag`
- **Purpose**: Indexes the synthetic SOP corpus and asset maintenance history. Serves retrieval requests.
- **Interface**: HTTP `/retrieve` with query, filter (asset ID, doc type), and top-k. Returns chunks with stable source IDs.
- **Implementation**: Qdrant or pgvector in the PoC. Embedding model runs in the same container or in `small-inference`.
- **State**: Persistent volume holding the index.

### `tool-inventory`
- **Purpose**: MCP server exposing the mock parts inventory.
- **Interface**: MCP. Tools: `inventory.check_part(part_id)`, `inventory.reserve_part(part_id, quantity, capability_token)`.
- **Implementation**: Python MCP server, SQLite backing store.
- **State**: Persistent. Holds the synthetic inventory.
- **Authority**: Every mutating call requires a capability token in the request. The tool validates the token with the broker before acting. Reads may also require tokens depending on scope policy (decide in ADR).

### `tool-workorder`
- **Purpose**: MCP server exposing the mock work-order system.
- **Interface**: MCP. Tools: `workorder.draft(...)`, `workorder.route_for_approval(workorder_id, capability_token)`. Explicitly no `workorder.dispatch` — that requires human action outside the system.
- **Implementation**: Python MCP server, SQLite backing store.
- **State**: Persistent.

### `tool-history`
- **Purpose**: MCP server exposing read access to the asset maintenance history. Distinct from `rag` because the history is structured (events) while RAG operates on unstructured SOPs.
- **Interface**: MCP. Tools: `history.for_asset(asset_id, since, capability_token)`.
- **Implementation**: Python MCP server, SQLite backing store.

### `broker`
- **Purpose**: Issues, validates, and audits capability tokens. The enforcement point for all authority.
- **Interface**: HTTP. Endpoints:
  - `POST /capabilities/request` — agent requests a capability. Broker decides, issues or denies, logs.
  - `POST /capabilities/validate` — tool adapter validates a presented token. Broker confirms or rejects.
  - `GET /capabilities/{id}` — read a capability record (for audit/replay tools).
- **Implementation**: Python. SQLite for the capability ledger.
- **State**: The capability ledger is part of the audit substrate. Append-only.
- **Note**: The broker is small, inspectable, and the part the auditor will scrutinize most. Keep it that way.

### `orchestrator`
- **Purpose**: Drives the scenario. Receives an incoming ticket, calls the small model, queries RAG, calls the mid model, requests capabilities from the broker, calls tool adapters with the granted tokens, emits events to the audit bus.
- **Interface**: HTTP `/tickets` for submitting a ticket. Returns a run ID. `GET /runs/{id}` returns the current state.
- **Implementation**: Python. Stateless except for in-flight runs.
- **State**: In-memory run state. The durable record is in the audit log.

### `audit-bus`
- **Purpose**: Receives all events from all components. Persists them. Serves replay queries.
- **Interface**: NATS subject `castellan.events` for ingest. HTTP `/events` for querying with filters. HTTP `/replay/{run_id}` for narrative replay.
- **Implementation**: NATS for ingest. Python consumer that writes to SQLite with hash chaining.
- **State**: The append-only event store. The most important persistent thing in the system.

### `demo-ui`
- **Purpose**: Minimal web view that shows the demo running. Not pretty, but legible — shows each step, the model outputs, the capability requests and decisions, the final work order, and the replay view.
- **Interface**: HTTP.
- **Implementation**: A single static HTML + small JS file backed by `orchestrator` and `audit-bus` HTTP endpoints. No framework necessary.

## Topology

A single `docker-compose.yml` brings up all services on one host. A shared overlay network connects them. Persistent volumes for `rag`, the tool adapters, the broker ledger, and the audit store.

```
  ┌──────────────┐
  │   demo-ui    │
  └──────┬───────┘
         │ HTTP
  ┌──────▼───────┐    HTTP    ┌──────────────┐
  │ orchestrator ├───────────►│    broker    │
  └──┬─┬─┬─┬─────┘            └──────┬───────┘
     │ │ │ │                          │ validate
     │ │ │ │  MCP/HTTP                ▼
     │ │ │ └────────────────► tool-inventory ─────┐
     │ │ │                                         │
     │ │ └──────────────────► tool-workorder ─────┤
     │ │                                           │
     │ └────────────────────► tool-history ───────┤
     │                                             │
     │  OpenAI HTTP                                │
     ├──────────────► small-inference              │
     ├──────────────► mid-inference                │
     └──────────────► rag                          │
                                                   │
                  NATS                             │
  all components ──────► audit-bus ◄───────────────┘
                                events
```

## Interface conventions

- **Model calls** use OpenAI-compatible HTTP. Wrapper in the orchestrator handles the request/response and emits an audit event.
- **Tool calls** use MCP. The orchestrator is the MCP client. Each tool adapter is an MCP server.
- **Capability tokens** are opaque strings (UUID4) carried in MCP tool call arguments as a `capability_token` field. The token is not a JWT in the PoC — the broker is the only authority, and tool adapters validate against the broker. (JWT or signed-token swap is a possible future ADR.)
- **Audit events** are JSON, published to NATS subject `castellan.events`. Every service that produces events uses a shared schema (see `event-schema.md`).

## What lives where

| Concern | Lives in |
|---|---|
| Which model gets called for what | `orchestrator` |
| Which tool gets called for what | `orchestrator` |
| Whether an action is allowed | `broker` |
| Why an action was allowed or denied | `broker`, logged to audit |
| What the model saw | `audit-bus` (full prompt + retrieved context) |
| What the tool returned | `audit-bus` |
| The synthetic plant data | `rag`, `tool-inventory`, `tool-workorder`, `tool-history` |
| The narrative replay | `audit-bus` reconstructs it |

## What is deliberately not in the architecture

- A dedicated "policy engine." Policy lives in the broker. If the broker grows complex enough to warrant a policy DSL, that's a future ADR.
- A message queue beyond NATS. The audit bus is the only async path.
- A user management system. The PoC has a single hardcoded operator identity.
- A dedicated secrets manager. Capability tokens are short-lived and held in memory.
- gRPC, GraphQL, or any non-HTTP/non-MCP protocol. Boring is the point.

## Dev vs demo configurations

A single Compose file with profiles. The `dev` profile uses CPU-only inference with small models and skips GPU requirements. The `demo` profile uses GPU-attached inference if available. Both produce the same audit log shape.

## Open questions deferred to ADRs

- Specific embedding model for RAG.
- Whether read-only tool calls require capability tokens or only mutating ones.
- Whether the broker's policy is hardcoded for the PoC or driven by a config file.
- Specific small and mid model picks (will depend on dev hardware).
