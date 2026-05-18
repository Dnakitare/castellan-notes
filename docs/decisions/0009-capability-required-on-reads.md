# 0009 — Capability required on every tool call, reads included

Date: 2026-05-17
Status: accepted

## Decision

Every call to a tool adapter (`tool-workorder`, `tool-inventory`,
`tool-history`) requires a capability_token, including read-only operations
like `inventory.check_part` and `history.for_asset`. The token is validated
with the broker and consumed (single-use) just as it is for mutating
operations. The `rag` service is not a tool adapter and is exempt.

## Context

Architecture spec said: "Every mutating call requires a capability token...
Reads may also require tokens depending on scope policy (decide in ADR)."
This is that ADR. The relevant constraints:

- **Auditor uniformity.** Every tool action lands in the chain with a
  `capability_id` linking back to a principal. If reads are ungated, the
  auditor cannot answer "who looked at the inventory of BEARING-X42 at
  10:43:02 last Tuesday." With capability-on-reads, the answer is in the
  chain.
- **Operator-role policy already allows reads.** `OPERATOR_ALLOWED` in
  `src/broker/policy.py` already grants `inventory.part/read` and
  `history.asset/read`. Requiring capabilities doesn't restrict what the
  operator can do; it just makes the access attributable.
- **Cost is essentially zero.** Both validate and consume are local HTTP
  calls against the broker's SQLite ledger. Two calls per read adds
  milliseconds.
- **rag is conceptually different.** It serves retrieval over unstructured
  text against a synthetic public corpus. It's an information service, not
  a system-of-record proxy. The architecture spec treats `rag` and
  `tool-history` as distinct exactly because the former is unstructured
  SOPs and the latter is the structured event record. Putting the
  capability gate on `tool-history` and not `rag` preserves that
  distinction.

## Alternatives considered

- **Reads ungated; only mutations require capabilities.** Simpler and
  cheaper per call. Rejected because it leaves the auditor unable to
  attribute reads, and creates a two-tier mental model ("which tool calls
  need a capability again?") that bites at slice step 8 when the second
  tool adapter lands.
- **Reads gated but not consumed (validate-only).** Skips the single-use
  cost for read paths so an operator can poll inventory cheaply. Rejected
  because it introduces a third capability lifecycle ("granted but
  endlessly re-usable for reads") that the broker's ledger doesn't model
  and that the auditor would have to special-case.

## Consequences

Easier:
- One mental model for tool adapters: "every call validates, every call
  consumes, every call emits `tool.invoked`."
- The capability ledger is a complete record of who looked at what, when.
- ADR scope of "what needs a capability" is settled for the rest of the
  slice; tool-inventory and tool-history (step 8) inherit this directly.

Harder:
- An operator running a long inspection (multiple inventory reads) needs to
  request multiple capabilities. The orchestrator can batch this by
  requesting a parent capability with the right scope and issuing
  per-read children, but that's a future concern; the slice has one read
  per run at most.
- More events in the chain. Acceptable — the chain is the point.

To revisit:
- If the operator workflow demands many reads per operation and the audit
  noise becomes a real concern, consider a read-only capability lifecycle
  (granted, expires, never consumed) or a read-batch capability that
  permits N reads of a given scope before consumption.
