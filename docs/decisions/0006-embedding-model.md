# 0006 — Embedding model: nomic-embed-text-v1.5 (Q8_0), served by llama.cpp

Date: 2026-05-17
Status: accepted

## Decision

The `embeddings` container runs `ghcr.io/ggml-org/llama.cpp:server` in
`--embedding` mode against `nomic-embed-text-v1.5.Q8_0.gguf` (~139 MB) from
`https://huggingface.co/nomic-ai/nomic-embed-text-v1.5-GGUF`. It exposes
OpenAI-compatible `/v1/embeddings` on port 11436 inside the compose network.
Embedding dimensionality is 768. The init-container pattern reused from the
small/mid tiers handles the fetch.

## Context

The RAG service needs an embedder for both ingest (one-shot at startup) and
query time (one per `/retrieve` call). Constraints:

- Must run alongside small (~2.5 GB) and mid (~5 GB) inference containers.
  Budget for the embedder is on the order of hundreds of MB, not several GB.
- Must serve over HTTP so the `rag` service is decoupled from the embedding
  implementation, in the same way the orchestrator is decoupled from
  small/mid via OpenAI-compatible HTTP.
- Reusing the existing llama.cpp container pattern (image, init-container,
  model volume, marker file) keeps the architecture story tight: every model
  in the system is a GGUF served by llama-server, period. No second model
  framework to explain to an auditor.

Candidates considered:

| Model                         | Q8_0 size | Dims | Notes |
|-------------------------------|----------:|-----:|-------|
| nomic-embed-text-v1.5         |    139 MB | 768  | strong on retrieval benchmarks (MTEB) |
| BAAI/bge-small-en-v1.5        |    ~40 MB | 384  | smaller; weaker on long-context |
| all-MiniLM-L6-v2              |    ~24 MB | 384  | ubiquitous baseline, dated |

Nomic is the best overall on MTEB at this size class and natively supports
the 768-dim space we want for the synthetic corpus. The 139 MB cost is
trivial next to the 1.8 + 4.4 GB already in flight for the chat tiers.

The Q8_0 quant is chosen over Q4_K_M (~80 MB) because embedding quality is
more sensitive to quantization than chat output, and the 60 MB difference
doesn't matter.

## Alternatives considered

- **Sentence-transformers in-process inside the `rag` container.** Pulls in
  `torch` (~700 MB wheel) and a per-language tokenizer stack. Heavier than
  the entire llama.cpp container plus the GGUF combined, and uses a
  different model-serving paradigm than the rest of the system. Rejected.
- **Reuse `small-inference` for embeddings.** llama-server loads exactly one
  model per process; sharing would require swapping models in/out per call
  or running two processes in one container. Rejected as both fragile and
  surprising. Architecture spec mentioned this option but the constraint
  wasn't visible at spec-time.
- **OpenAI / Voyage / external embedding API.** Adds a network hop, an API
  key, and a vendor dependency. The PoC's premise is on-prem; rejected.
- **BGE-small or MiniLM at 384 dims.** Lighter, but at this corpus size the
  storage saving is meaningless and retrieval quality is visibly worse on
  the kinds of SOP queries we'll demo.

## Consequences

Easier:
- One model-serving framework across small / mid / embeddings.
- Same init-container fetch script, same `.url` marker, same resume logic.
- 139 MB extra disk is noise next to the chat tiers.

Harder:
- `/v1/embeddings` returns the OpenAI shape: `{"data": [{"embedding": [...]}], ...}`. The `rag` service needs to parse this, but it's a few lines.
- Embedding inference on CPU adds ~100-200 ms per chunk on first call (warm-up). Subsequent calls amortize. Acceptable for the slice's tiny corpus.

To revisit:
- A 1024-dim or larger embedder (e.g., `mxbai-embed-large`) if retrieval quality on the synthetic SOPs looks weak.
- Batched embeddings — llama-server supports batch — if ingest grows past a few hundred chunks.
- Sharing one embedder container across multiple `rag` collections if the system ever has more than one.
