# 0013 — Parts list is derived from a hardcoded per-asset-class table

Date: 2026-05-18
Status: accepted

## Decision

The orchestrator derives the parts list for the inventory step from a
hardcoded `PARTS_FOR_ASSET_ID` table keyed by the ticket's `asset_id`
(per ADR 0010 the asset id is always explicit in the ticket). For the
slice and the seeded demo, that table is

```python
PARTS_FOR_ASSET_ID = {
    "PUMP-3B": [{"part_id": "BEARING-X42", "quantity": 1}],
}
```

The orchestrator does not attempt to extract part identifiers from the
mid model's diagnosis text or from the RAG-retrieved SOP body. Keying off
`asset_id` rather than the classifier's `asset_class` also avoids one more
classifier-output dependency: the inventory step proceeds correctly even
when the classifier writes the asset class slightly differently from run
to run (`"equipment"` vs `"centrifugal_pump"` vs `"pump"`).

## Context

The inventory step needs an answer to: "what parts does this repair
require?" The SOP body has the answer ("...replacement BEARING-X42, two
units per pump, one unit sufficient for single-position replacement...")
but it's in prose. The candidate extraction approaches:

1. **Hardcoded per-asset-class table (this ADR).** Tiny, deterministic,
   auditor-readable.
2. **Mid model returns structured parts as part of its diagnosis.** Forces
   a JSON schema on the mid response and re-introduces the parsing-and-
   fallback logic we removed via ADR 0012.
3. **Pattern-match the SOP chunk for known part IDs.** Fragile — depends
   on which chunk the embedder ranks first; depends on the SOP author
   formatting part IDs the same way every time.
4. **A second small model call dedicated to "extract parts."** A whole new
   model invocation per run, more cost, another point of failure, and the
   auditor surface gains nothing.

The same reasoning that locked in ADR 0010 (explicit `asset_id` in the
ticket) applies here: the slice is about authority and audit, not about
extraction quality. The wrong parts would route the wrong capability
request through the broker for the wrong reservation — clean audit chain,
clean broker decision, wrong outcome. That is the failure mode the trust
story exists to prevent.

A real maintenance system would have a structured SOP database (the SOP
authoring step is upstream of any LLM); the orchestrator can hit that
when it lands. Until then, the table is honest about the limitation.

## Alternatives considered

- **Structured parts field on the SOP, surfaced via a new RAG endpoint or
  a separate `tool-sop` adapter.** The cleanest forward path, but adds a
  service. Defer until there's more than one SOP and the table approach
  visibly creaks.
- **Mid model in structured-output mode.** Forces grammar-constrained
  decoding, doable with llama.cpp's `grammar` parameter. Real consideration
  for the next iteration of the mid prompt; out of scope here.
- **Hardcode in the seed.json rather than the orchestrator code.** Moves
  the mapping into data, which is nicer for non-engineer demo tweaks. The
  orchestrator would load it on startup. Reasonable; rejected for the slice
  because the orchestrator is the consumer and putting the table next to
  the consumer keeps the code locally legible.

## Consequences

Easier:
- Inventory step is deterministic for the seeded demo: every run for a
  centrifugal pump reserves BEARING-X42 qty 1, every audit-replay reads
  the same on every run.
- No new failure mode introduced by parts extraction; the only ways the
  inventory step can fail are real authority/availability concerns (broker
  denies, stock insufficient), both of which are the demo-worthy refusal
  cases.
- The "what would change to support a real plant" answer is short: replace
  the table lookup with a query against the structured SOP store.

Harder:
- The mid model's diagnosis text is for human consumption only; it doesn't
  feed the action. That gap is real but it's a quality concern about the
  diagnosis, not a trust concern about the agent.
- A new asset class added to the seed corpus requires a code change to the
  table. Documented; accept until the system has more than one asset class.

To revisit:
- When/if a real structured SOP database lands, replace this with a
  `tool-sop` adapter call. Supersede this ADR.
- If the mid prompt's quality matters enough to drive structured outputs,
  consider grammar-constrained decoding for the parts list specifically
  (keep the diagnosis prose for human reading).
