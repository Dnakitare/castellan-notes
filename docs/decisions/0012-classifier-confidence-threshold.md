# 0012 — Classifier confidence threshold drives the escalate path

Date: 2026-05-18
Status: accepted

## Decision

The orchestrator parses the classifier's `confidence` field out of the small
tier's JSON response. If `confidence < CLASSIFIER_CONFIDENCE_MIN` (default
`0.7`), or if the classifier returns unparseable output or omits the field
entirely, the orchestrator emits

  `agent.refused` reason_code=`low_classifier_confidence`

and closes the run with `outcome=escalated`. No retrieve, no diagnose, no
broker calls, no tool calls. The work passes to human triage.

## Context

`scenario.md` is explicit that the small classifier is "deliberately humble
— when confidence is low, it routes to 'needs human triage' rather than
guessing." `auditor-view.md` lists "low-confidence classification →
escalation, not action" as a failure mode the PoC must demonstrate visibly
in the audit chain. Without this check the orchestrator would happily
proceed to ask for capabilities and drive tools on a ticket the system did
not actually understand — defeating the entire trust story.

A confidence threshold has the right shape for this PoC:

- The check is on the orchestrator side (one branch in `runner.py`), not
  hidden in a model prompt.
- The decision is auditable: the threshold is a config value, the
  classifier's `confidence` lands in the `model.invoked` event payload,
  and the `agent.refused` event records the comparison.
- The threshold is one knob: an operator can dial it up (more cautious,
  more escalations) or down (more autonomous, more risk) and the change is
  visible in config.

`0.7` matches the example in `event-schema.md`'s `agent.refused` payload
(`"reason_text": "Classifier confidence 0.41 below threshold 0.7"`). The
default is environment-overridable for demo tuning.

## Alternatives considered

- **Ensemble or self-consistency check.** Run the classifier multiple times
  and use disagreement as a confidence signal. More expensive, more lines
  of code, and the auditor surface (single confidence number) gets more
  abstract. Rejected for the slice.
- **Have the mid model "second-opinion" the classification.** Two model
  tiers chained. Loses the "humble small tier escalates fast" property —
  by the time the mid runs, we've spent the latency budget that escalation
  is supposed to save.
- **Always proceed; let the broker refuse on policy mismatch.** Defeats
  scenario.md's humility design. Also wastes mid inference + RAG on a
  ticket the system shouldn't act on.
- **Hidden in the orchestrator's prompt: "if you're not sure, say so."**
  Inconsistent enforcement (depends on the model behaving), unattributable
  in audit (the auditor sees "model said the right thing this time" rather
  than "system enforced the threshold").

## Consequences

Easier:
- Auditor reads the chain top to bottom and sees the system refused to act
  with the classifier's own numeric confidence in evidence.
- The escalate path is a single branch; no new services, no new event
  types beyond what's already in the schema.
- Tuning happens in env, not code; deployments can move the threshold for
  their risk appetite.
- Smoke target for the refusal demo is small and stable.

Harder:
- The classifier has to honestly report low confidence on ambiguous input.
  Qwen 2.5 3B is biased toward high confidence regardless; the system
  prompt has to push it to be honest, and the threshold may need tuning
  per model.
- A classifier that returns malformed JSON also gets treated as
  low-confidence (reason_text="malformed classifier output"). That's the
  right call but means a parsing bug masquerades as an escalation. The
  reason_text disambiguates in the audit chain.

To revisit:
- Move from a single threshold to per-asset-class thresholds when more
  asset types arrive.
- Add a second-look path (auto-retry classifier with a stricter system
  prompt) before escalation, if confidence-near-threshold runs turn out to
  be common.

## The actual contract

`Config.classifier_confidence_min` (env `CLASSIFIER_CONFIDENCE_MIN`,
default `0.7`).

Refusal triggers:

- `parsed["confidence"]` missing or not a number → escalate with reason_text
  `"classifier did not report confidence; treating as low"`
- `float(parsed["confidence"]) < threshold` → escalate with reason_text
  `"classifier confidence {c:.2f} below threshold {t:.2f}"`

`agent.refused` payload:

```json
{
  "reason_code": "low_classifier_confidence",
  "reason_text": "classifier confidence 0.41 below threshold 0.70",
  "context": {"confidence": 0.41, "threshold": 0.70, "classification": "..."}
}
```

`run.completed` payload then carries `outcome=escalated`,
`final_workorder_id=null`, `summary="Escalated to human triage: classifier
confidence below threshold."`.
