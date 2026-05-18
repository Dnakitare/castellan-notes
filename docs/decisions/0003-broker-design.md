# 0003 — Broker design

Date: 2026-05-16
Status: accepted

## Decision

The broker is a single Python process exposing a small HTTP surface for
capability lifecycle (request, validate, consume, read) backed by a SQLite
ledger. The decision logic is a pure function over a request and a ledger
snapshot, separated from any I/O. All requests — granted and denied — are
persisted and audited. Audit events are published to the same NATS JetStream
subject the audit-bus consumes from, via a new shared `AuditPublisher`
abstraction in `castellan_core` so that broker, orchestrator, and tool
adapters all use the same publish path.

The full refusal taxonomy from `authority-model.md` is implemented now at
step 3 of the vertical slice, not deferred to step 2 of the slice progression.
Building all five refusal cases (unknown principal, out-of-role action, scope
creep, expired parent, constraint violation) when the policy code is fresh is
much cheaper than retrofitting them later, and it keeps slice-2's refusal
demos to wiring rather than broker work.

Capabilities are single-use, enforced at `consume`. `validate` is the gate
that tool adapters call before performing work; `consume` is what they call
after the work succeeds.

## Context

The broker is the only enforcement point for the authority model — every
other component is allowed to be wrong about who can do what as long as the
broker is correct. That makes broker size and shape an auditor-facing
concern: `auditor-view.md` says explicitly that the trust story collapses to
"trust the broker and the audit substrate." The broker therefore needs:

- A small, inspectable surface (HTTP, a handful of endpoints).
- Decision logic that can be read top-to-bottom without chasing references.
- A ledger that is the source of truth for capability state and is queryable
  for replay.
- Every decision — including refusals — visible in the audit chain.

The publisher abstraction is shared because the same code shape will recur in
every service that emits events (orchestrator, three tool adapters, both
inference wrappers). Keeping it in `castellan_core` means there is one
implementation of canonical-JSON-serialize-then-publish and one place to
change behavior (e.g., switching from JetStream publish to a queued/batched
publish later).

## Alternatives considered

- **Per-service audit publisher copy/paste.** Smaller blast radius from a bad change, but immediately stale — every service drifts. Rejected; this is exactly the kind of cross-cutting code that benefits from a shared library.
- **Defer the full refusal taxonomy to a later slice.** Tempting given step 3's spec scope. Rejected because the refusal taxonomy is the broker's main job — building only the happy path here would force a near-total rewrite at slice-2.
- **Single-use enforced at `validate`.** Would make validate+consume the same op and eliminate the validate-vs-consume race window. Rejected because it changes the contract in `authority-model.md` and removes the tool-adapter's ability to abort between "I'm allowed to do this" and "I did it." The race is acceptable for the PoC (single orchestrator per run); a future ADR can revisit if multi-agent flows arrive.
- **Pydantic-validated stored scope.** Cleaner than `scope_json` text column. Rejected for now to keep the SQLite schema obvious and migration-free; the scope is parsed back into a dict on every read anyway.
- **Combine `capability.requested` and `capability.decision` into one event.** Simpler audit chain. Rejected: separating them lets the auditor see the request as the agent phrased it before seeing the broker's verdict, which is the entire point of having both fields per the event schema.

## Consequences

Easier:
- Refusal paths in slice-2 become "compose an ambiguous ticket and watch the
  broker say no" rather than a broker rewrite.
- The `AuditPublisher` interface means tool adapters and the orchestrator can
  be written without re-inventing NATS publish.
- Policy is testable in isolation; full refusal coverage in `tests/` without
  any container infrastructure.

Harder:
- A bug in `AuditPublisher` affects every producer at once. Mitigation: it is
  thin and has its own tests.
- The validate→consume window is a known small race. Single-orchestrator PoC
  is unaffected; flagged for revisit if concurrency grows.
- `castellan_core` now optionally pulls in `nats-py` when `AuditPublisher` is
  imported. The import is lazy so stubs and the canonical-JSON utilities have
  no runtime nats dependency.

To revisit:
- Move principals registry and role policy out of code into a config file
  when more than one role exists.
- Signed capability tokens (JWT) for distributed validation instead of every
  validate call hitting the broker. Today the broker is the sole authority.
- Atomic validate-and-consume primitive if multi-agent or parallel flows
  arrive.
