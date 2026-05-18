# 0005 — Mid-tier model: Qwen 2.5 7B Instruct (Q4_K_M)

Date: 2026-05-16
Status: accepted

## Decision

The mid-tier inference container (`mid-inference`) is served by
`Qwen2.5-7B-Instruct-Q4_K_M.gguf`, pulled from
`https://huggingface.co/bartowski/Qwen2.5-7B-Instruct-GGUF/resolve/main/Qwen2.5-7B-Instruct-Q4_K_M.gguf`
(~4.4 GB). Context length is set to 8192 in `llama-server`.

## Context

The mid tier's job is reasoning over retrieved context: take the small
tier's classification plus the asset's maintenance history (via RAG, arriving
at slice step 5) plus the relevant SOPs, and produce a diagnostic narrative
with a proposed procedure that cites its sources. This needs enough capacity
to hold several thousand tokens of context and produce a coherent multi-step
response, but it has to fit on a developer laptop alongside the 3B small
tier and the rest of the Compose stack.

Candidates:

| Model                              | Q4_K_M size | RAM in use | Notes |
|------------------------------------|------------:|-----------:|-------|
| Qwen 2.5 7B Instruct               |     4.36 GB |       ~6 GB | strong reasoning at modest cost |
| Qwen 2.5 14B Instruct              |     8.37 GB |      ~10 GB | better reasoning; needs 16 GB+ RAM |
| Llama 3.1 8B Instruct              |     4.92 GB |       ~6 GB | comparable to Qwen 7B |
| Mistral Nemo 12B Instruct (Q4_K_M) |     7.0 GB  |       ~9 GB | very strong; cross-family with 3B small |

7B is the default that lets a typical 16 GB dev machine run the full Compose
topology (NATS + audit-bus + broker + small + mid + future RAG) without
swapping. 14B is an easy URL swap when running on hardware with the
headroom — the ADR is structured so it can be superseded by a one-file
ADR 0005a if the user upgrades hardware or moves to a dedicated demo box.

Mirror choice is bartowski, for the same reasons as ADR 0004 (consistency,
public, well-tested). The official `Qwen/Qwen2.5-7B-Instruct-GGUF` repo is
HTTP 404 at the time of this ADR.

## Alternatives considered

- **Qwen 2.5 14B Instruct.** First-choice for quality. Rejected as the default because it requires ~10 GB resident which would push the full stack uncomfortable on a 16 GB laptop. URL swap is trivial when hardware permits.
- **Llama 3.1 8B Instruct.** Comparable quality, same size class. Rejected for the family-consistency reason in ADR 0004 — running one model family across tiers keeps the audit trail's `model_name` field readable as "small/mid Qwen 2.5" rather than "Qwen 3B + Llama 8B."
- **Mistral Nemo 12B.** Excellent at long context, but it's a different family, doesn't reuse the Qwen prompt template, and the bigger size buys little for a demo whose narrative is "tier each model to the job."
- **A 32k-context Qwen variant.** Not needed yet — the slice's prompts plus retrieved SOPs and history will sit comfortably under 8k tokens. Reconsider at slice step 5 if RAG context grows.

## Consequences

Easier:
- Fits the typical 16 GB dev machine; the full Compose stack runs without swapping.
- Same family + same prompt template as the small tier, so prompt iteration in slice steps 5–7 doesn't need per-tier prompt forks.
- Context length 8192 covers the planned prompts (classification echo + retrieved SOPs + history snippet) with margin.

Harder:
- 7B is noticeably less capable than 14B for multi-step reasoning. For the demo's diagnostic narrative this should still read coherently; if it doesn't, the URL swap to 14B is the lever.
- ~4.4 GB first-fetch download on a fresh checkout; mitigated by the volume cache.

To revisit:
- If the demo narrative produced at slice step 6 reads thin, supersede this ADR by pinning to 14B (assuming 16+ GB free RAM on the demo machine).
- If multiple operators run the demo in parallel on one host, 14B's RAM cost makes 7B the right call regardless of single-user feasibility.
