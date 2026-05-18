# 0007 — Vector store: Qdrant

Date: 2026-05-17
Status: accepted

## Decision

The `rag` service stores chunk embeddings in Qdrant (`qdrant/qdrant:latest`),
running as a separate container in the compose network. Collection name is
`castellan_chunks`, distance metric is cosine, vector size matches the
embedder's output (768 with nomic-embed-text-v1.5 per ADR 0006). The
collection is created idempotently on `rag` startup; ingest is idempotent
on Qdrant point count vs seed chunk count (skip if equal).

## Context

The RAG service needs to (a) hold embeddings keyed by stable
`source_id`/`chunk_id`, (b) support nearest-neighbour search with metadata
filters (asset_id, doc_type), and (c) be durable across restarts so we
don't re-embed the seed corpus every time. The two realistic candidates:

- **Qdrant**: single container, REST API on 6333 plus gRPC on 6334, payload
  filter language built in, durable file storage. ~150 MB image.
- **pgvector**: postgres extension. Brings the entire postgres operational
  surface — users, roles, migrations, connection pooling — for what amounts
  to one table.

Qdrant fits the "boring single-purpose container" rule from the architecture
spec. pgvector would make more sense if the project already had Postgres
running for something else; it does not.

## Alternatives considered

- **pgvector.** Heavier operational footprint, schema management overhead, no advantage on this corpus size. Rejected unless/until we need transactional semantics linking embeddings to other relational data.
- **Chroma.** Lighter than Qdrant, popular for prototypes. Rejected because its persistence story has been inconsistent across releases and the auditor narrative is "trust the stores you're already comfortable with" — Qdrant is more established in production deployments.
- **In-memory FAISS + a JSON dump.** Avoids a container, but loses metadata filtering, persistence, and the architecture story collapses into "the rag service holds the index in its own memory," which is exactly the kind of thing the auditor would push back on.

## Consequences

Easier:
- Filter expressions (`{"must": [{"key": "asset_id", "match": {"value": "PUMP-3B"}}]}`) handle the architecture spec's "filter by asset ID, doc type" requirement directly.
- Qdrant collection inspection (`GET /collections/{name}`) is a useful smoke-test surface.
- Persistent volume means we re-embed only when the seed corpus changes.

Harder:
- One more container (~150 MB image, ~100 MB resident). Acceptable inside a 12 GB Docker VM.
- Qdrant's API requires explicit collection creation before upsert; handled at startup.

To revisit:
- If we ever need transactional embedding + relational data (e.g., capability ledger pointing into the same store), revisit pgvector.
- Quantization of embeddings (`scalar`/`product` quant in Qdrant) if storage ever matters; not relevant at PoC corpus sizes.
