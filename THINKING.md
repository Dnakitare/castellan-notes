# Thinking

Running log of decisions and what I learned building Castellan. Not a changelog. A record of the reasoning behind the shippable moments.

## 2026-06-07, evaluating the two refusal surfaces

### What this is, and what it isn't

An eval harness for the two places Castellan can refuse: the authority broker withholding a capability, and the classifier gate declining to act on a low-confidence ticket. Two harnesses, because the broker is deterministic policy and the classifier sits on a 3B model, and they fail in different ways. In scope: refusal correctness at both surfaces, scored against a labelled golden set. Not in scope: the mid-tier diagnosis model. I did not measure whether the diagnosis is any good, only whether the system acts or refuses when it should. That gap is deliberate and named, not an oversight.

### Why this approach

The project's whole claim is that you trust the broker and the audit log, not the model. A claim like that is worth what its evidence is worth. So I wanted the refusals measured, not asserted.

The authority harness scores the broker's policy function directly instead of going through the HTTP endpoint. The function is pure, so the eval is deterministic, runs in under a second, and doubles as a CI gate. For a policy decision point, 100% is the expected result, not a brag. The value is that it covers the whole refusal taxonomy including the parent-composition cases, and it turns any future regression into a red test.

The classifier harness drives the real ADR 0012 gate. It reuses the production system prompt and the production parser instead of copying them, so a passing run means the real gate behaves, not a lookalike. I fed it three kinds of ticket: clear, obviously vague, and adversarial. The adversarial ones sound urgent or technical but state nothing concrete. That is the case that matters, because a model that mistakes fluent phrasing for certainty fails there.

It did. A 40% false-act rate at the production threshold, all of it adversarial. The model gave 0.9 to 1.0 confidence to tickets with no symptom in them. I checked whether sampling could rescue it. At deterministic decoding the three samples per ticket were identical, so there is nothing to harvest from disagreement. The failure is systematic, not noise.

So I wrote a hardened prompt that declares tone to be not-evidence and anchors confidence to a concrete observable. Same tickets, same threshold, same model. False-act fell to 20% with no regression on clear or vague. The real change was the distribution. Under the baseline the adversarial tickets scored on top of the clear ones, which is why no threshold could split them. Under the hardened prompt the clear tickets sit at 0.90 and the adversarial drop to 0.40 to 0.70. The classes became separable.

### What's fragile

- The golden set is sixteen tickets. Enough to surface a failure and measure a fix, not enough to trust a single number. The threshold that scores 100% on this set is tuned on sixteen examples. I report it with that caveat and lean on the two claims that survive a larger set: false-act halved at the unchanged threshold, and the distribution went from overlapping to separable.
- The hardened prompt is a candidate, measured in the eval, not yet adopted into the production classifier prompt. Adopting it is a separate decision with its own regression check.
- The mid-tier diagnosis is unevaluated. Anyone reading the numbers should know the eval covers refusal, not diagnostic quality.

### What I learned

The instructive failure is the one the fix did not close. Two adversarial tickets survived at exactly 0.70. One was pure urgency. The other was "the flux capacitor is reading 1.21 gigawatts, classic sign of dilithium fracture." The hardened prompt rewards a concrete reading attached to a part, and that sentence is, in form, a reading attached to a part. The model can verify the shape of a symptom. It cannot verify that the symptom is real, because a 3B classifier does not know a flux capacitor is fiction. That is the line between sounding right and being right, and a system prompt does not move it. Closing it needs a different mechanism: checking claimed observables against a real asset registry, or routing the boundary cases to the mid tier. The harness is now the instrument that would score either one.

The second thing: a prompt fix is not only a number, it is a change to a distribution. The baseline and hardened false-act rates differ by twenty points, but the more useful fact is that the hardened prompt made two classes separable that were not before. I would have missed that if I had read only the headline metric and not the per-ticket confidences.
