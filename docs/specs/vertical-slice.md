# Vertical Slice

This document defines the smallest end-to-end thing worth building first. The point is to exercise every architectural boundary — model, RAG, broker, tool, audit — on the simplest possible happy path, before fleshing out the full scenario in `scenario.md`.

If this slice works, the architecture is real. If it doesn't, more scope won't save it.

## The slice

A single hardcoded ticket flows through the system:

> Ticket: `"Pump 3B making grinding noise on startup, vibration noticeable, started this morning."`
> Operator: `op-42`

The system:

1. Classifies the ticket using the **small model** (severity, asset class, subsystem). One model call. One `model.invoked` event.
2. Retrieves the asset's history and any matching SOPs from **RAG**. One `rag.queried` event.
3. Produces a diagnostic narrative and proposed procedure using the **mid model**, given the classification + retrieved context. One `model.invoked` event.
4. Requests a single capability from the **broker** to draft a work order. One `capability.requested` event, one `capability.decision` event (granted).
5. Calls **tool-workorder** to draft (not route, not dispatch) the work order. One `capability.validated`, one `capability.consumed`, one `tool.invoked` event.
6. Emits `run.completed` with outcome `drafted`.
7. The demo UI shows the run; the audit replay tool can reconstruct it.

That's the slice. No inventory check. No human approval routing. No refusal paths. One synthetic asset (`PUMP-3B`), one synthetic SOP, one synthetic history record. Hardcoded operator. Hardcoded broker policy.

## What this slice intentionally proves

- The two model tiers run locally and are reachable via OpenAI-compatible HTTP.
- RAG indexes and serves a tiny synthetic corpus.
- The orchestrator can sequence: classify → retrieve → diagnose → request capability → invoke tool.
- The broker issues and validates a capability against a real tool adapter.
- The audit bus receives events from every component, chains them, and serves a replay.
- Everything runs under `docker compose up` on a single workstation.

## What this slice intentionally does not prove (yet)

- Multiple model tiers cooperating on tricky cases. Add later.
- Refusal paths. Add in the next slice.
- Inventory check / parts availability. Add in the next slice.
- Multi-step capability composition (parent/child capabilities). Add in the next slice.
- Multiple operators or roles. Add later if at all in the PoC.
- A real demo narrative the buyer would watch. The full scenario is what gets demoed; this slice is the spine.

## Build order for this slice

The natural order for Claude Code, one per session (or thereabouts):

1. **Skeleton.** Repo layout, `docker-compose.yml` with empty service stubs, a `make demo` target that brings them up, a shared Python package for event schemas.
2. **Audit bus.** NATS in Compose, the Python consumer that writes to SQLite with hash chaining, the `/events` and `/replay/{run_id}` and `/verify` endpoints, a smoke test that emits a few events and verifies the chain.
3. **Broker (minimal).** Capability ledger, `/capabilities/request`, `/capabilities/validate`, `/capabilities/consume`. Hardcoded operator-role policy. Emits the four capability events.
4. **Inference servers.** `llama.cpp` server containers for the small and mid models, each with its model GGUF fetched into a volume by an init container (per `docs/decisions/0001-inference-runtime.md`). Skeleton uses SmolLM2-135M for both tiers; this step swaps in real per-tier model URLs. A tiny script in the orchestrator that calls each with a canned prompt and emits `model.invoked`.
5. **RAG.** Qdrant or pgvector container. Ingest script for the synthetic corpus (one SOP, one history record to start). `/retrieve` endpoint. Emits `rag.queried`.
6. **Tool-workorder.** MCP server with a `draft` tool. SQLite backing. Validates capability tokens with the broker. Emits `tool.invoked`.
7. **Orchestrator (slice).** The end-to-end sequence for the hardcoded ticket. Emits `run.started`, drives the pipeline, emits `run.completed`.
8. **Demo UI (minimal).** Single static HTML page that POSTs the hardcoded ticket and renders the replay. No framework.
9. **End-to-end smoke test.** `make demo` brings everything up, submits the ticket, asserts the run completes with `outcome: drafted`, asserts the audit chain verifies clean.

Each step's deliverable is: the component running in Compose, a smoke test passing, an updated `handoff.md`.

## Synthetic data for the slice

A single file `data/seed.json` containing:

```json
{
  "principals": [
    {"id": "operator:op-42", "role": "operator", "active": true}
  ],
  "assets": [
    {"id": "PUMP-3B", "class": "centrifugal_pump", "location": "Building 2, Line 1"}
  ],
  "sops": [
    {
      "id": "SOP-PUMP-001",
      "title": "Bearing inspection and replacement, centrifugal pumps",
      "asset_class": "centrifugal_pump",
      "body": "...a few paragraphs of realistic-sounding SOP text covering inspection, parts list, and procedure..."
    }
  ],
  "history": [
    {
      "id": "HIST-PUMP-3B-001",
      "asset_id": "PUMP-3B",
      "date": "2024-11-15",
      "event": "Bearing replacement (BEARING-X42) following vibration alert. 4h labor. Returned to service same day."
    }
  ],
  "inventory": [
    {"part_id": "BEARING-X42", "on_hand": 3, "asset_classes": ["centrifugal_pump"]}
  ]
}
```

An ingest script run during `docker compose up` loads this into the appropriate stores.

## Acceptance for "slice complete"

- `make demo` succeeds on a clean checkout.
- The hardcoded ticket runs end to end in under 30 seconds (small model fast, mid model up to 15s acceptable on dev hardware).
- The replay endpoint returns a coherent narrative of the run.
- `GET /verify` on the audit bus returns clean.
- The minimal UI shows the sequence visually.
- The README has one paragraph explaining what was demonstrated and how to read the replay.

Once these hold, the next slice adds the refusal path and the inventory step.
