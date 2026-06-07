# Castellan — notes

*Delegated authority for local AI agents, applied to industrial maintenance triage. A specification and the reasoning behind it.*

## The problem

Most AI agent systems shown to industrial buyers fail the same way. The demo shows the model doing something impressive, the buyer asks "what stops it from doing the wrong thing," and the answer is some combination of "the model is good" and "we have guardrails." That is not an answer a maintenance superintendent, a safety officer, or a regulator can sign off on.

The interesting question, then, is not whether the model is good. It is whether the *system around the model* is shaped so that a model going off the rails cannot translate into action. Authority is the right abstraction for this. Models can suggest; only authority can act.

## The thesis

The auditor's takeaway, when reviewing a system like this, must be:

> *"I do not have to trust the AI. I have to trust the authority broker and the audit substrate. Those are small, inspectable, and behave like things I already know how to evaluate."*

That sentence is the north star. Everything else follows from it.

## The shape of the answer

Castellan is a proof-of-concept for that idea, applied to the most legible authority artifact in industrial work: the maintenance work order. The architecture has four parts worth naming.

- **A tiered local model stack.** Small model classifies, mid model reasons, agent orchestrates. *Local* because regulated industrial buyers will not send plant data to a hyperscaler. *Tiered* because a single 70B model doing everything tells the buyer the demonstrator does not understand their cost or trust structure.
- **An authority broker.** Every action — every read, every write, every tool call — requires a scoped, time-limited, single-use capability issued by the broker against a stated principal. The broker is the enforcement point. An agent that goes off the rails cannot exceed what the broker will grant.
- **A hash-chained, append-only audit substrate.** Every model call, retrieval, tool call, and authority decision is logged into a tamper-evident chain. The log alone is sufficient to reconstruct what happened. The orchestrator writes to it; it does not own it.
- **Refusal as a first-class outcome.** Low classifier confidence, missing context, denied capability, parts unavailable — each terminates the run cleanly with a logged reason. The auditor can find the non-actions as easily as the actions.

## The scenario

A free-text maintenance ticket arrives. ("Pump 3B making grinding noise on startup, vibration noticeable, started this morning.") Castellan classifies it, retrieves the asset's history and the relevant SOP, drafts a diagnostic narrative with cited references, checks parts availability, reserves the parts, drafts the work order, and routes it for human approval. It does not dispatch work autonomously. A second run, on a deliberately ambiguous ticket, shows the system refusing to act and escalating — visibly, in the same UI, with the same audit chain.

The maintenance domain is the vehicle. The point is the trust and architecture story.

## What is in this repository

- [`docs/specs/`](docs/specs/) — six specifications: scenario, auditor view, architecture, authority model, event schema, vertical slice. The source of truth the build follows.
- [`docs/decisions/`](docs/decisions/) — thirteen architectural decision records. Each is two paragraphs: what was decided, why, what was rejected.
- [`docs/evals/`](docs/evals/) — how the two refusal surfaces are measured. The broker scores 0% false grants across the full taxonomy; the classifier gate scores a 40% false-act rate on adversarial confident-but-empty tickets, which is the point: it shows exactly where the model layer fails and why the authority layer underneath it matters.

The specs predate the code. Where the implementation and a spec disagreed, one of them was wrong and the disagreement was resolved before the session ended.

## What is not in this repository

The working implementation — Python services, Docker Compose topology, native llama.cpp inference, end-to-end demo, ~144 passing tests — is private. This repository is the public face: the thesis and the reasoning, not the demo.

## Status

Proof-of-concept, May 2026. The implementation runs end to end on a single workstation: an Apple Silicon laptop with native llama.cpp processes a complete ticket — classify → retrieve → diagnose → inventory check → parts reservation → work order draft, twenty audit events, clean hash chain — in roughly 70 seconds.

The point is not the latency. It is that every one of those twenty events is something an auditor can read, replay, and verify without trusting the demonstrator.

---

*Built solo, spec-first, as a study in what it would take for delegated authority in local agent systems to clear an industrial buyer's bar.*

*Contact: [dnakitare@gmail.com](mailto:dnakitare@gmail.com)*
