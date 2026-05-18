# Event Schema

Every event in Castellan conforms to a common envelope. The envelope is small and stable; the payload varies by event type. All events flow through NATS subject `castellan.events` and are persisted by `audit-bus` to an append-only store with hash chaining.

## Envelope (every event has this)

```json
{
  "event_id": "uuid4",
  "sequence": 12345,
  "timestamp": "2026-05-17T01:23:45.678Z",
  "run_id": "uuid4-or-null",
  "producer": "orchestrator | broker | tool-inventory | tool-workorder | tool-history | small-inference | mid-inference | rag",
  "event_type": "see below",
  "schema_version": "1",
  "prev_event_hash": "sha256-hex-of-previous-event-bytes",
  "payload": { /* varies by event_type */ }
}
```

- `event_id` is unique per event.
- `sequence` is monotonically increasing across the entire system, assigned by `audit-bus` on ingest.
- `run_id` ties events belonging to the same scenario run together. Null for system events outside a run.
- `producer` is the component that emitted the event.
- `prev_event_hash` is the SHA-256 of the canonical-JSON serialization of the immediately preceding event (by sequence). The first event has `prev_event_hash` = 64 zeros. Tampering with any past event breaks the chain.
- `schema_version` is currently `"1"`. Bumps go in an ADR.

## Event types and payloads

### `run.started`
Emitted by orchestrator when a new ticket arrives.
```json
{
  "ticket_id": "TKT-2026-0517-001",
  "ticket_text": "Pump 3B making grinding noise on startup...",
  "principal": "operator:op-42",
  "received_at": "2026-05-17T01:23:45.678Z"
}
```

### `run.completed`
Emitted by orchestrator when a run terminates (successfully, escalated, or denied).
```json
{
  "outcome": "drafted | escalated | denied",
  "final_workorder_id": "WO-2026-0517-001 | null",
  "summary": "Work order drafted and routed for approval | Escalated: low classifier confidence | Denied: parts unavailable"
}
```

### `model.invoked`
Emitted by orchestrator each time a model is called. The full prompt and response are captured.
```json
{
  "model_tier": "small | mid",
  "model_name": "qwen2.5:3b",
  "purpose": "classification | diagnosis | procedure_drafting",
  "prompt": "...full prompt text...",
  "response": "...full response text...",
  "tokens_in": 412,
  "tokens_out": 187,
  "latency_ms": 743
}
```

### `rag.queried`
Emitted by orchestrator each time RAG is queried.
```json
{
  "query": "PUMP-3B bearing maintenance procedure",
  "filters": {"asset_id": "PUMP-3B", "doc_type": "sop"},
  "top_k": 5,
  "results": [
    {"source_id": "SOP-PUMP-001", "chunk_id": "SOP-PUMP-001#3", "score": 0.87},
    {"source_id": "HIST-PUMP-3B-2024-11", "chunk_id": "HIST-PUMP-3B-2024-11#1", "score": 0.81}
  ]
}
```

### `capability.requested`
Emitted by broker when a capability request arrives.
```json
{
  "capability_id": "uuid4",
  "principal": "operator:op-42",
  "requesting_agent": "orchestrator",
  "scope": {
    "resource": "inventory.part",
    "action": "reserve",
    "constraints": {"part_id": "BEARING-X42", "quantity_max": 1}
  },
  "justification": "Reserving bearing for proposed maintenance on PUMP-3B per procedure SOP-PUMP-001",
  "parent_capability_id": null,
  "requested_expires_in_seconds": 60
}
```

### `capability.decision`
Emitted by broker after evaluating a request.
```json
{
  "capability_id": "uuid4-matches-request",
  "decision": "granted | denied",
  "decision_reasoning": "Principal in active operator role; scope within role; constraints reasonable for ticket context.",
  "issued_at": "2026-05-17T01:23:46.000Z",
  "expires_at": "2026-05-17T01:24:46.000Z"
}
```

### `capability.validated`
Emitted by broker when a tool adapter validates a presented token.
```json
{
  "capability_id": "uuid4",
  "presented_by": "tool-inventory",
  "expected_scope": {"resource": "inventory.part", "action": "reserve"},
  "result": "valid | expired | scope_mismatch | already_consumed | unknown"
}
```

### `capability.consumed`
Emitted by broker when a single-use capability is consumed.
```json
{
  "capability_id": "uuid4",
  "consumed_by": "tool-inventory",
  "consumed_at": "2026-05-17T01:23:47.123Z"
}
```

### `tool.invoked`
Emitted by a tool adapter when it performs a tool call.
```json
{
  "tool_name": "inventory.reserve_part",
  "tool_arguments": {"part_id": "BEARING-X42", "quantity": 1},
  "capability_id": "uuid4",
  "result": {"reservation_id": "RES-2026-0517-001", "quantity_reserved": 1},
  "success": true,
  "latency_ms": 22
}
```

### `agent.refused`
Emitted by orchestrator when it declines to proceed.
```json
{
  "reason_code": "low_classifier_confidence | empty_retrieval | capability_denied | parts_unavailable | model_refused",
  "reason_text": "Classifier confidence 0.41 below threshold 0.7; routing to human triage.",
  "context": {"classifier_output": {...}, "threshold": 0.7}
}
```

### `system.error`
Emitted by any component when something unexpected happens.
```json
{
  "error_type": "string",
  "error_text": "string",
  "component_state": {...}
}
```

## Canonical JSON for hashing

`prev_event_hash` is computed over a canonical serialization to ensure determinism:
- UTF-8 encoded
- Keys sorted lexicographically at every object level
- No insignificant whitespace
- Numbers in shortest unambiguous form

The audit-bus is responsible for this serialization. Producers send events as ordinary JSON; the bus canonicalizes, assigns the sequence and prev-hash, and persists.

## Storage layout (PoC)

A single SQLite database `audit.db` with one table:

```sql
CREATE TABLE events (
  sequence    INTEGER PRIMARY KEY,
  event_id    TEXT NOT NULL UNIQUE,
  timestamp   TEXT NOT NULL,
  run_id      TEXT,
  producer    TEXT NOT NULL,
  event_type  TEXT NOT NULL,
  prev_hash   TEXT NOT NULL,
  this_hash   TEXT NOT NULL,
  body_json   TEXT NOT NULL
);

CREATE INDEX idx_events_run ON events(run_id);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_producer ON events(producer);
CREATE INDEX idx_events_timestamp ON events(timestamp);
```

`this_hash` is SHA-256 of the canonical-JSON of the full envelope (excluding `this_hash` itself, naturally).

## Query API

`audit-bus` exposes HTTP endpoints:

- `GET /events?run_id=...&type=...&producer=...&since=...&until=...&limit=...` — filtered list.
- `GET /events/{sequence}` — single event by sequence.
- `GET /replay/{run_id}` — returns the events for a run in sequence order, with a `narrative` field that renders the run as human-readable text.
- `GET /verify` — walks the entire chain and reports any hash mismatch.

## What is deliberately not in the schema

- Encryption at rest. The PoC operates on a single host with a single trust domain.
- Multi-tenant fields. One tenant.
- PII redaction policy. Synthetic data only; no real PII flows through the PoC.
- External signing (e.g., Sigstore/Rekor). Possible later ADR.
