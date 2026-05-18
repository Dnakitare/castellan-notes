# Auditor View

This document describes what an auditor, compliance reviewer, or risk officer sees when they evaluate Castellan. It is written from their perspective deliberately, because every other architectural choice in this project is downstream of "what does this person need in order to sign off."

The auditor is not the buyer's engineer. They are the person the buyer's engineer has to convince. They are skeptical, time-constrained, and burned by past AI demos that showed only the happy path.

## What the auditor must be able to answer

After reviewing Castellan, the auditor must be able to answer, without help from the demonstrator:

1. **Who acted?** For any action the system took, who was the principal — the human or role on whose behalf the action was taken?
2. **What authority did they have?** What scoped capability authorized the action? Where did that capability come from? When does it expire?
3. **Why was it granted?** What request did the agent make to the broker, and what was the broker's reasoning for granting (or denying)?
4. **What did the model see?** For any model-produced output that influenced an action, what was the input — the prompt, the retrieved context, the tool results?
5. **Can I replay it?** Can the auditor reconstruct the full sequence of events leading to any action, in order, with timestamps?
6. **What didn't happen?** Can the auditor see the cases where the system refused to act, and why?

If the auditor cannot answer any one of these, the system has failed its purpose regardless of demo polish.

## What the auditor sees, concretely

**An append-only event log.** Every event has a stable schema (see `event-schema.md`), a monotonically increasing sequence number, a wall-clock timestamp, and a cryptographic chain (each event references the hash of the previous event, so tampering is detectable). The log is queryable by principal, by capability ID, by asset, by time range, and by event type.

**An authority ledger.** Every capability ever issued by the broker — granted or denied — with the request that prompted it, the scope, the lifetime, and the decision reasoning. Capabilities are first-class objects, not implicit tokens scattered through the code.

**A replay tool.** A command-line tool that, given a work order ID or a time range, reconstructs the full sequence of events as a human-readable narrative. Same data as the raw log, presented for review.

**Refusal records.** When the system declines to act — low classifier confidence, missing context, denied capability, parts unavailable — that decision is logged with the same rigor as a positive action. The auditor must be able to find these as easily as they find actions taken.

## What the auditor explicitly does not need to trust

- **The model.** The model's output is treated as a suggestion, not an action. Actions happen via capabilities issued by the broker. A model that hallucinates a procedure cannot cause that procedure to be executed without a capability being requested, granted, and logged.
- **The agent's good behavior.** The agent's authority is bounded by what the broker will grant. The broker is the enforcement point. An agent that goes off the rails cannot exceed its broker-granted capabilities.
- **The demonstrator's narration.** Everything the demonstrator says about the system can be verified against the audit log without their cooperation.

## Properties the audit substrate must have

- **Append-only.** No event is ever modified or deleted in place. Corrections are new events that reference the original.
- **Tamper-evident.** Each event includes the hash of the previous event in its chain. Removing or modifying a past event breaks the chain and is detectable.
- **Complete.** Every model call, every retrieval, every tool call, every capability request and decision, every refusal. If it influenced behavior, it is logged.
- **Replayable.** The log alone is sufficient to reconstruct what happened. No reliance on in-memory state or external logs.
- **Independent of the orchestrator.** The orchestrator writes to the audit bus; it does not own the audit store. The audit log survives an orchestrator crash or compromise.

## Failure modes the auditor explicitly looks for

These should be visible in the PoC, on purpose:

1. **Low-confidence classification → escalation, not action.** The auditor sees the system stop and route to human review when the classifier is unsure.
2. **Missing or stale asset history → diagnosis declines.** If RAG returns nothing relevant, the mid model is instructed to say so rather than guess. The audit log shows the empty retrieval and the model's refusal.
3. **Capability denied → action does not occur.** If the broker denies a capability request (out of scope, expired principal, etc.), the agent does not proceed. The denial is logged with reasoning.
4. **Parts unavailable → no draft work order.** The agent does not draft a work order that cannot be fulfilled. It logs the gap and escalates.
5. **Authority does not include dispatch.** Even with all other steps successful, the agent has no capability to dispatch work autonomously. A human approval step is required and visible.

## What good looks like from the auditor's seat

The auditor's takeaway, ideally, is: *"I do not have to trust the AI. I have to trust the authority broker and the audit substrate. Those are small, inspectable, and behave like things I already know how to evaluate."*

That sentence is the project's north star.
