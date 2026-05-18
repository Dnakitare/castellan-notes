# Scenario — Maintenance Work Order Triage

## What the demo does

A maintenance ticket arrives in a fictional industrial plant — free text, the kind a shift operator would type or dictate. ("Pump 3B making grinding noise on startup, vibration noticeable, started this morning.")

Castellan processes it through a tiered local AI stack:

1. A **small classifier model** reads the ticket and determines severity tier, asset class, and likely subsystem. It runs fast, locally, and is deliberately humble — when confidence is low, it routes to "needs human triage" rather than guessing.
2. A **mid-tier reasoning model** receives the classified ticket along with the asset's maintenance history (retrieved via RAG over a synthetic but realistic corpus) and produces a diagnostic narrative plus a proposed maintenance procedure with cited references to past work and SOPs.
3. An **agent orchestrator** takes the proposed procedure and, on the operator's behalf, performs the bounded actions a work-order system would expect: check parts availability against a mock inventory, reserve required parts if available, draft the work order, and route it to a human approver. It does **not** dispatch work autonomously.

Every step is governed by the **authority broker**. The agent does not have ambient access to the inventory system or the work-order system; it requests a scoped, time-limited capability from the broker for each action, with the request including who the action is on behalf of and why. The broker logs the decision. Every model call, retrieval, tool call, and authority decision lands in an append-only audit log that can be replayed.

When the system encounters a situation it cannot handle confidently — ambiguous ticket, missing history, parts unavailable, procedure that requires authority the agent doesn't have — it stops and escalates rather than improvising. This refusal behavior is shown explicitly in the demo, not hidden.

## Why this scenario

Maintenance work orders are a real authority artifact in industrial environments. They authorize physical work on physical assets, and the question of who authorized what, on whose behalf, with what scope, is something maintenance organizations already care about — it shows up in incident investigations, in regulatory audits, and in insurance disputes. This makes the authority model legible to the buyer without requiring the buyer to first accept that authority modeling matters.

It also exercises every tier of the local LLM stack (small, mid, agentic) and makes the case for each — the small model's job is genuinely a small-model job, the mid model's job genuinely needs reasoning over context, and the agent's job genuinely needs orchestration across systems. A buyer who sees one 70B model doing everything will correctly assume the demonstrator does not understand the cost or trust structure of their world.

Finally, it is simulable solo. The asset history, the SOP corpus, the inventory, and the work-order system can all be synthetic, and the synthetic data does not have to be perfect to make the architectural point.

## What the demo is not

It is not a real CMMS replacement. It is not integrated with any real industrial protocol. It does not claim to diagnose equipment correctly in general — only to show that *if* it diagnoses, *how* it acts on that diagnosis is governed and auditable. The demo is about the trust and architecture story; the maintenance domain is the vehicle.

## The narrative beats of the demo

A single demo run, end to end, in roughly this order:

1. **Ticket arrives.** Free text shown on screen.
2. **Classification.** Small model produces severity, asset class, subsystem. Confidence shown.
3. **Retrieval.** Asset history and relevant SOPs retrieved. Sources cited.
4. **Diagnosis.** Mid model produces narrative diagnosis with references to retrieved material.
5. **Procedure proposed.** Specific steps and parts list.
6. **Authority request.** Agent requests scoped capability to check inventory. Broker grants, logs.
7. **Inventory check.** Parts available; agent requests capability to reserve. Broker grants, logs.
8. **Work order drafted.** Agent requests capability to draft (not dispatch). Broker grants, logs.
9. **Routed for approval.** Human-in-the-loop step shown explicitly. No autonomous dispatch.
10. **Audit replay.** Reviewer scrolls through the full audit trail — every model call, every retrieval, every authority decision, replayable.

A second run, on a deliberately ambiguous ticket, shows the refusal path: classifier flags low confidence, agent escalates without acting, audit log shows the non-action.

## Out of scope for the PoC

- Real-time streaming of alarms or telemetry
- Voice input
- Multi-site federation
- Fine-tuning models on plant data
- Any actual industrial protocol
- User authentication beyond a static operator identity
- Pretty UI — legible terminal output or a minimal web view is sufficient
