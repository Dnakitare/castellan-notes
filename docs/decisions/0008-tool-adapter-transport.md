# 0008 — Tool adapters use HTTP, not MCP

Date: 2026-05-17
Status: accepted

## Decision

The tool adapters (`tool-workorder`, `tool-inventory`, `tool-history`) speak
plain HTTP, not Model Context Protocol. Each tool gets its own URL of the
shape `POST /tools/{tool_name}` with body
`{"capability_token": "...", "arguments": {...}, "run_id": "...", "presented_by": "..."}`.
The body shape mirrors MCP's `tools/call` payload so a future swap to MCP
is mechanical — change the dispatcher and the transport, the contract stays
identical.

## Context

The architecture spec calls these services "MCP servers." The auditor-facing
guarantee is the *boundary* — each tool is its own process with its own
state and its own capability validation — not the wire format. MCP would
add:

- Tool discovery via `tools/list` handshakes.
- The JSON-RPC request/response framing.
- A stdio-or-HTTP transport choice the orchestrator has to negotiate.

None of those add anything the auditor reads. The trust story is identical
under either transport: "the broker validated this capability before the
tool acted; the tool logged its action to the audit chain."

HTTP keeps the slice's surface readable:

- One URL per tool ⇒ trivially curl-able for smoke and demo.
- One Pydantic body schema per tool ⇒ FastAPI gives validation, OpenAPI
  docs, and 422s for malformed input for free.
- Same `httpx` client every other service already uses for inference and
  broker calls — no second protocol library.

## Alternatives considered

- **Full MCP over stdio.** Closest to "real MCP" usage. Rejected: stdio
  transport doesn't fit Docker Compose's network-first topology, and the
  orchestrator would need a process manager just to talk to the tools.
- **MCP over HTTP (streamable HTTP).** Less invasive than stdio. Rejected
  for now because the protocol still requires capability handshakes (`tools/list`,
  `initialize`) that add nothing for a fixed-set-of-tools PoC. The body
  shape we use is forward-compatible with MCP's `tools/call`.
- **gRPC.** Strong typing and streaming, but a third protocol in the system
  alongside HTTP and NATS. Rejected as scope creep.

## Consequences

Easier:
- One protocol across orchestrator, broker, inference, tools — only HTTP.
- Tool adapter Dockerfiles look identical to the broker's Dockerfile.
- Tests use FastAPI's `TestClient` exactly like the broker + rag + audit-bus
  tests already do.

Harder:
- An MCP-aware client cannot speak to our tool adapters today. That client
  doesn't exist in the system, so no current cost.
- If we later want LLM-driven tool selection where the model lists tools and
  picks one dynamically, we'll need to add a `tools/list` endpoint and (likely)
  switch to MCP. That refactor is bounded: the body shape was chosen to make
  it small.

To revisit:
- If/when the orchestrator integrates an MCP-native LLM agent runtime,
  re-evaluate and supersede this ADR.

## Body shape (the actual contract)

Request:

```json
POST /tools/workorder.draft
{
  "capability_token": "<uuid>",
  "arguments": {"asset_id": "PUMP-3B", "summary": "...", "severity": "medium"},
  "run_id": "<uuid|null>",
  "presented_by": "orchestrator"
}
```

Response (200):

```json
{
  "workorder_id": "WO-2026-0517-001",
  "status": "draft",
  "created_at": "2026-05-17T01:23:47.123Z"
}
```

Error responses use FastAPI's `HTTPException` with status codes:

- `400` — scope_mismatch or other validate failure that the tool can name
- `403` — capability denied (decision was `denied` at request time)
- `404` — unknown capability_id
- `409` — already_consumed
- `410` — expired
- `422` — body shape failed Pydantic validation

Every outcome — success or failure — produces a `tool.invoked` event in the
audit chain, with `success` set accordingly.
