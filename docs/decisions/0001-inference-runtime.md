# 0001 — Inference runtime: llama.cpp server, not Ollama

Date: 2026-05-16
Status: accepted

## Decision

Use `ghcr.io/ggml-org/llama.cpp:server` as the inference runtime for both the
small and mid tiers. Each tier runs its own container with its own GGUF model
file mounted from a named volume. Models are pre-fetched into the volume by a
one-shot init container that runs before the server.

## Context

The original tech default in `CLAUDE.md` was Ollama, picked for its
model-management ergonomics (`ollama pull <name>`). Concretely on disk:

| Option         | Runtime image | Per-model file (3B Q4) | Skeleton-day pull |
|----------------|---------------|------------------------|-------------------|
| Ollama         | ~700 MB–1 GB  | ~2 GB                  | ~1 GB (image only) |
| llama.cpp:server | 130 MB      | ~2 GB                  | ~130 MB + tiny smoke model |

The skeleton step doesn't exercise inference yet; the inference containers sit
idle. Paying 1 GB for two idle Ollama copies is wasteful. The architecture
spec already says "OpenAI-compatible HTTP" is the only contract — anything
serving `/v1/chat/completions` fits, so the runtime is genuinely swappable
without touching anything but the inference services.

llama.cpp's official server image exposes the same OpenAI-compatible endpoint,
ships as a single C++ binary, and is ~6× smaller. The trade is losing
`ollama pull`-style model management — we replace it with an init container
that downloads a specified GGUF URL into the model volume on first boot.

## Alternatives considered

- **Keep Ollama.** Larger image, but `ollama pull` is genuinely nicer than wrangling GGUF URLs. Rejected for skeleton-phase disk cost; the model-management convenience does not justify the 6× image overhead when we're pulling models from HuggingFace either way.
- **Stub inference entirely until step 4.** Lightest option (~0 MB), but defers the runtime choice instead of making it, and leaves the inference plumbing untested in the skeleton.
- **vLLM, TGI, LocalAI.** All heavier than Ollama. Wrong direction.
- **Roll our own FastAPI wrapper over `transformers`.** Brittle, slow, no quantization story without extra work. Rejected.

## Consequences

Easier:
- Smaller image footprint; faster `make demo` first-run on a clean box.
- Direct llama.cpp tuning surface (context length, threads, KV-cache size) is exposed via flags rather than hidden behind Ollama's modelfile.
- No daemon-managed model registry; the GGUF file in the volume *is* the model.

Harder:
- No `ollama pull`. We fetch GGUFs via curl in an init container; model selection becomes "pick the right HuggingFace GGUF URL."
- No automatic model swapping — one server, one model. Already true at the architecture level (small-inference and mid-inference are separate containers), so this costs nothing in practice.
- Quantization choice is now explicit. We'll record the chosen GGUF variant per tier in a follow-up ADR at vertical-slice step 4.

To revisit at step 4:
- Specific small-tier GGUF (likely Qwen 2.5 3B Instruct Q4_K_M or Llama 3.2 3B Instruct Q4_K_M).
- Specific mid-tier GGUF (Qwen 2.5 14B Q4_K_M if it fits dev hardware, else Qwen 2.5 7B or Llama 3.1 8B).
- Context length per tier.

## Skeleton-phase model

Both tiers point at `SmolLM2-135M-Instruct-Q4_K_M` (~92 MB). It produces
incoherent output, which is exactly what we want for a skeleton smoke test —
the topology is real, the response is real JSON conforming to OpenAI's
schema, and we are not tempted to confuse the smoke test with capability.
