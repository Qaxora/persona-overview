# ADR-0024: Memory embeddings are computed at ingest, in the background

- **Status:** Accepted
- **Date:** 2026-09-29
- **Decision code:** D-24

## Context

Semantic search compares a question vector with memory vectors. Computing memory vectors at query time would mean embedding every memory of the user on every question — impossible in cost and latency.

## Decision

- **Write time:** each memory's embedding is computed once, asynchronously after capture (it never delays the `201` response), and stored in `memory_embeddings` with the embedding model name and dimension (ADR-0010).
- **Read time:** only the question is embedded; pgvector returns the nearest memories, always filtered by `UserId`.
- **What is embedded:** the raw memory text is the baseline layer — it exists even when analysis fails, so search never depends on analysis success. Item-level embeddings (e.g. of `Representation`) may be added later as a second layer.
- **Model change:** changing the embedding model triggers re-computation; rows record which model produced them.

## Consequences

- An embedding job joins the capture pipeline (its own status can follow ADR-0003 conventions).
- Embeddings are derived data: deleted with the memory (ADR-0005, ADR-0019) and treated as special-category data (ADR-0020).
