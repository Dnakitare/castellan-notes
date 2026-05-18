# 0010 — Ticket payload carries an explicit `asset_id`

Date: 2026-05-17
Status: accepted

## Decision

The ticket payload that enters the orchestrator at `POST /tickets` includes
an explicit `asset_id` field. The orchestrator does not attempt to extract
the asset identity from the free-text body of the ticket. If a ticket
arrives without `asset_id`, the orchestrator escalates rather than guesses.

For the slice and the demo, the seeded ticket carries
`"asset_id": "PUMP-3B"` alongside the free-text body
`"Pump 3B making grinding noise on startup..."`.

## Context

The seeded scenario has the operator typing a ticket that mentions "Pump
3B." A plausible (and tempting) design is "let the orchestrator parse the
text and pull out the asset identifier." It would feel modern and would
make the demo look more autonomous.

Two reasons not to:

1. **It is a quality risk at the most auditor-visible step.** If the
   orchestrator hallucinates an asset id — confuses PUMP-3B with PUMP-3D,
   picks an asset class that matches but a wrong instance, etc. — the
   capability that gets requested is *for the wrong asset*, every
   downstream action is correctly authorized for the wrong thing, and the
   audit chain looks completely clean. That is the worst possible failure
   mode for an authority story: a system that's airtight about who can do
   what, applied to the wrong what.

2. **Real maintenance systems already structure tickets this way.** A CMMS
   ticket carries an asset reference; the free text describes the symptom.
   The "operator types it" framing in the scenario is shorthand for "the
   operator selects an asset, then describes the issue." We can preserve
   the demo affordance without bending the data model.

The orchestrator's job is to apply tiered reasoning *given* an asset; it is
not to identify the asset. That responsibility belongs to whatever produced
the ticket (CMMS form, voice-to-ticket transcriber, alarm router, etc.).

## Alternatives considered

- **NLP-driven extraction in the orchestrator (small model).** The
  classifier could be prompted to also extract an asset_id. Rejected for
  the failure-mode reason above; small models are not reliable on this
  task and the cost of a wrong extraction is high.
- **Extraction with a verifier (small extracts → orchestrator looks up in
  the asset registry → confirms exact match).** Plausible but adds an
  asset registry and a verification step for a problem we do not actually
  have. Defer until the demo narrative demands "operator types raw text
  with no structured affordance."
- **No `asset_id`, infer from `asset_class` + history.** Falls apart the
  moment there is more than one centrifugal pump in the plant.

## Consequences

Easier:
- Orchestrator code is shorter and more honest about what it does.
- Refusal path is obvious: missing `asset_id` → escalate.
- Capability requests carry the correct `asset_id` constraint by
  construction.

Harder:
- The demo loses the "magic" of typing raw text and watching the system
  figure out which asset. Mitigation: the demo can frame the input as a
  CMMS ticket with both fields, which is what it would actually be in a
  real deployment.
- A future "voice to ticket" or "alarm to ticket" feature has to do the
  extraction itself upstream of the orchestrator, not inside it.

To revisit:
- When/if a demo asks for unstructured text input, supersede this ADR with
  one describing the extraction-with-verifier pattern.
