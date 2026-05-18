# 0004 — Small-tier model: Qwen 2.5 3B Instruct (Q4_K_M)

Date: 2026-05-16
Status: accepted

## Decision

The small-tier inference container (`small-inference`) is served by
`Qwen2.5-3B-Instruct-Q4_K_M.gguf`, pulled from
`https://huggingface.co/bartowski/Qwen2.5-3B-Instruct-GGUF/resolve/main/Qwen2.5-3B-Instruct-Q4_K_M.gguf`
(~1.8 GB). Context length is set to 4096 in `llama-server`.

The init container's `fetch-model.sh` is extended to keep a `.url` marker
next to `model.gguf`. If the configured `MODEL_URL` no longer matches the
stored marker, the script re-downloads. This lets a URL change propagate
through `docker compose up` without manual volume cleanup.

## Context

The small tier's job in the scenario is fast, humble classification:
severity tier, asset class, subsystem. The model must run on a developer
laptop in CPU mode without delaying the demo, and it should be honest about
low confidence rather than confabulate.

Candidates:

| Model                          | Q4_K_M size | RAM | Instruction-following |
|--------------------------------|------------:|----:|-----------------------|
| Qwen 2.5 3B Instruct           |     1.80 GB | ~3 GB | strong, JSON-friendly |
| Llama 3.2 3B Instruct          |     2.02 GB | ~3 GB | strong, narrative tone |
| Phi-3.5 mini Instruct (3.8B)   |     2.30 GB | ~3.5 GB | strong reasoning, slightly larger |
| Mistral Nemo 12B (Q3_K_M)      |     5.5 GB | ~7 GB | overkill for the small tier |

Qwen 2.5 3B is well-regarded for tight JSON outputs, which matters for the
classifier whose output the orchestrator parses programmatically. It is also
the same family as the mid tier (ADR 0005), so the prompt style stays
consistent and the auditor sees one model family across the run.

Bartowski's mirror is chosen because the official
`unsloth/Qwen2.5-3B-Instruct-GGUF` repository requires HuggingFace auth
(401 from anonymous curl) — the same gating problem that forced moving off
`HuggingFaceTB/SmolLM2` at skeleton time. The `Qwen/Qwen2.5-3B-Instruct-GGUF`
official repo is public for the 3B but not for the 7B (ADR 0005), so picking
one consistent mirror (bartowski) for both tiers keeps the compose file
uniform.

## Alternatives considered

- **Llama 3.2 3B Instruct.** Equally capable on this task; chosen against because the prompt and the cross-tier consistency story are cleaner if the small and mid tiers are the same family.
- **`Qwen/Qwen2.5-3B-Instruct-GGUF`** (the official repo) — 200-OK and public, but only the 3B is officially distributed as GGUF; the 7B is not. Splitting mirrors across tiers is brittle and adds two ADRs' worth of churn the first time one mirror moves.
- **Q4_0 or Q5_K_M quantizations.** Q4_0 is smaller/faster but visibly less accurate on instruction-following; Q5_K_M is ~2.3 GB and barely more accurate at this scale. Q4_K_M is the standard sweet spot.
- **`unsloth/Qwen2.5-3B-Instruct-GGUF`.** Currently gated behind HF auth. Avoided; we picked the same mirror at skeleton time and hit the same wall.

## Consequences

Easier:
- One model family across both tiers; one prompt-template lineage; one tokenizer story.
- bartowski quantizations are very actively maintained and well-tested in the llama.cpp community.
- The `.url` marker means future URL changes (e.g., quantization swap, mirror move) just need a `compose up` — no manual volume removal.

Harder:
- Re-downloading 1.8 GB whenever the URL changes is non-zero. Mitigation: the marker only re-fetches if URLs differ, so re-pinning to the same URL is a no-op.
- Bartowski is a single uploader; if the repo goes away, we have to switch. Acceptable risk for the PoC.

To revisit:
- If demo prompts need the small tier to produce more structured tool-call-like outputs, consider a function-calling-tuned variant.
- Move to a smaller (1.5B) Qwen tier if dev laptop hits memory pressure with both small + mid loaded simultaneously.
