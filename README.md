# Qaxora Persona: Project Overview

**A personal memory system built on structured data, not chat history.**

This repository documents the design of Persona: its architecture, data flow and the engineering decisions behind it. The application code lives in a private repository and can be shared on request.

---

## What it does

Persona has three jobs: **understand → remember → retrieve.**

| | What happens | Example |
|---|---|---|
| **Capture** | The user writes or dictates a note in plain language. | *"Took Max to the vet today, paid 1,200 TL for his vaccines."* |
| **Analyse** | An LLM proposes structured items (people, pets, places, money, time, categories); the API validates and stores them. | pet → Max · category → pet health · amount → 1,200 TL · date → today |
| **Retrieve** | Questions are answered from stored memories, with the source memories attached. | *"How much did I spend on Max this year?"* → a sum computed in SQL, plus the memories it came from |

*(The example is illustrative; see the ADRs for the actual contracts.)*

Persona deliberately stops there. It does not send messages, make purchases or take any action outside the app.

## System design

```
Mobile (Expo) ──HTTP──▶ API (.NET) ──▶ PostgreSQL + pgvector   ◀── source of truth
                          │  one transaction: memory + idempotency record + outbox
                          ▼
                     outbox relay
                          ▼
                RabbitMQ (at-least-once)
      analysis.requested │            ▲ analysis.completed
                         ▼            │
                  AI service (Python / FastAPI) ── LLM (OpenAI / Ollama)
```

**Ownership is strict.** The API owns every table and every status change. The AI service has no database; it only turns requests into validated extraction results. Mobile holds no domain rules. Vectors, caches and model outputs are derived data that can be rebuilt from PostgreSQL.

Full description: [docs/architecture.md](docs/architecture.md)

## Engineering decisions

Each significant decision is written as an ADR before code depends on it. A selection:

| Problem | Decision | ADR |
|---|---|---|
| A mobile client retries a request after a timeout, possibly concurrently | `Idempotency-Key` + unique index, all writes in one transaction, first response replayed | [0001](docs/decisions/0001-concurrent-idempotent-requests-replay.md) |
| RabbitMQ may deliver the same message more than once | Inbox table keyed by `MessageId` in the API; best-effort cache in the AI service | [0006](docs/decisions/0006-duplicate-message-protection.md) |
| One malformed item in an LLM response used to discard the whole analysis | Validate item by item; keep valid items, log dropped ones, fail visibly if none survive | [0016](docs/decisions/0016-invalid-llm-items.md) |
| LLMs are unreliable at arithmetic and unsafe as SQL authors | The LLM produces a validated `QueryPlan`; .NET runs parameterised SQL / vector search and always adds the user filter | [0023](docs/decisions/0023-query-engine-architecture.md) |
| Semantic search needs vectors without slowing down capture | Embeddings computed once, in the background, at ingest; model name stored for re-computation | [0024](docs/decisions/0024-embedding-at-ingest.md) |
| Any memory may contain health or other sensitive data | All memory content handled as special-category data under KVKK | [0020](docs/decisions/0020-memory-content-is-special-category-data.md) |
| Small MVP scope vs. user trust and store/legal requirements | Deletion (memory and account) in the MVP; editing later | [0005](docs/decisions/0005-memory-correction-and-deletion.md) |

## Data and privacy

- Memory text never appears in production logs; tracing relies on correlation IDs.
- Every copy of memory text (database, analyses, embeddings) has a retention and deletion rule.
- Deleting a memory removes its derived data too.
- Encryption at rest and TLS between services are baseline requirements.

## Tech stack

| Layer | Technology |
|---|---|
| API | .NET, ASP.NET Core, EF Core, Npgsql, layered (DDD) structure |
| AI service | Python, FastAPI, Pydantic, aio-pika |
| Mobile | Expo / React Native, TypeScript |
| Data | PostgreSQL, pgvector |
| Messaging | RabbitMQ, outbox / inbox patterns, versioned JSON Schema contracts |
| Quality | xUnit + Testcontainers, pytest, GitHub Actions with separate API / AI / mobile jobs |

## Status

| Area | State |
|---|---|
| Idempotent capture (`201 Created`) | ✅ Implemented |
| AI extraction service (consume → extract → publish) | ✅ Implemented |
| Outbox relay, stored analysis runs, inbox dedupe | 🔧 In progress |
| Query engine (SQL + semantic, cited answers) | 🔧 In progress |
| Memory and account deletion, closed beta | 📋 Planned for MVP |

## Repository contents

```
README.md                 this overview
docs/architecture.md      current architecture
docs/decisions/           selected Architecture Decision Records
```

---

Built by [Emre Dal](https://github.com/byemredal) · [qaxora.com](https://qaxora.com)
