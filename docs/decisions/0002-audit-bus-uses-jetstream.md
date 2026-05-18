# 0002 — Audit bus uses NATS JetStream

Date: 2026-05-16
Status: accepted

## Decision

The audit bus consumes from a NATS JetStream subject, not core NATS. The
stream is `castellan_events` over subject `castellan.events`, retention is
`limits` with file storage on the NATS container's volume, and the bus runs
as a durable pull consumer named `audit-bus` with explicit acks. Messages are
acked only after the SQLite row is written and the chain is intact.

The bus also persists each event in its canonical envelope form (with
`sequence` and `prev_event_hash` filled in by the bus) directly into the
`body_json` column, so `this_hash = sha256(body_json bytes)` and chain
verification is a single byte-comparison per row.

## Context

`auditor-view.md` is explicit that the audit substrate must be complete —
every model call, retrieval, capability decision, refusal must land. If an
event is dropped because the bus was restarting at the moment of publish, the
chain has a hole and the entire trust story collapses. Core NATS is
fire-and-forget: published messages with no subscriber are gone.

JetStream gives us:

- **Durable subscription state** — if the bus crashes mid-batch, on restart it
  resumes from the last acked message.
- **Server-side persistence** — events published while the bus is down are
  buffered on the NATS container's volume and delivered on reconnect.
- **Explicit acks** — we ack only after the chain row commits, so a crash
  between receive and DB write redelivers the message.

The chain itself requires strict serialization: `prev_event_hash[N]` depends
on `this_hash[N-1]`, so the bus must process messages one at a time. A pull
consumer with max-in-flight = 1 fits naturally.

## Alternatives considered

- **Core NATS subject.** Lighter, but loses events across bus restarts. Rejected: violates the audit completeness property.
- **JetStream with multiple parallel consumers.** Would break chain ordering. Rejected.
- **Producer-assigned sequence/hash, bus is a passive sink.** Forces every producer to coordinate on a shared counter or accept gaps. Rejected: the bus being the single authority on sequence and chain is the simplest correct design.
- **Store the raw producer JSON and a separate canonical hash column.** Two sources of truth. Rejected; storing the canonical envelope itself in `body_json` makes `/verify` a pure byte comparison.

## Consequences

Easier:
- `/verify` is `sha256(row.body_json) == row.this_hash` and `row.prev_event_hash == prev_row.this_hash`. No re-serialization at verify time.
- Bus restart is safe: durable consumer resumes, idempotent stream creation on startup.
- Producers don't need to know their sequence number; they construct events with `sequence=None, prev_event_hash=None` and the bus fills them in.

Harder:
- Bus is a single writer; throughput is bounded by SQLite commit speed. Fine for PoC volumes.
- NATS container now needs JetStream enabled (already done in the skeleton: `nats -js -m 8222`) and a persistent volume for the stream. Adds `nats-data` named volume.
- `body_json` size carries the full envelope including payload, which can be large for `model.invoked` events containing full prompts and responses. Accepted for PoC; PII redaction is explicitly out of scope.

To revisit:
- Per-tenant streams if multi-tenancy ever enters scope.
- An external append-only log (Sigstore/Rekor) for off-system tamper evidence.
- Compression on `body_json` for large `model.invoked` payloads.
