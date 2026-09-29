# Persona Architecture

Current architecture of the Persona codebase (private repository). A selection of the decisions behind it is in [`decisions/`](decisions/README.md). `PS-xx` identifiers refer to items in the private issue tracker.

## 1. Shape of the system

A **modular monolith API** plus a **stateless AI worker**, talking over RabbitMQ with contract-tested messages. One repository, independently deployable services.

```
Mobile (Expo) ──HTTP──▶ API (.NET 10) ──▶ PostgreSQL (source of truth)
                          │  capture transaction: memory + idempotency + outbox
                          ▼
                    outbox relay (PS-06, pending)
                          ▼
                 RabbitMQ exchange persona.memory
        memory.analysis.requested │           ▲ memory.analysis.completed
                                  ▼           │
                          AI service (Python) ── LLM provider (OpenAI / Ollama)
```

| Component | Tech | Owns | Status |
|---|---|---|---|
| `apps/api` | .NET 10, ASP.NET Core, EF Core, Npgsql | All persistence, domain rules, statuses, user scope | Running |
| `apps/ai` | Python 3.11, FastAPI, aio-pika, Pydantic | LLM extraction; no database | Running |
| `apps/mobile` | Expo / React Native, TypeScript | UI only; no domain rules | Running (partly on mocks) |
| `apps/rag` | — | Reserved. Embeddings are computed by `apps/ai` and stored by the API (ADR-0010, ADR-0024) | Empty skeleton |
| PostgreSQL | `pgvector/pgvector:pg16` | Canonical memories and every derived table | Running (`infrastructure/docker`) |
| RabbitMQ | `rabbitmq:3-management` | Transport only | Running (`infrastructure/docker`) |
| Ollama | `ollama/ollama` or native install | Local LLM for development | Optional |

## 2. The API (`apps/api`)

DDD layers, feature-first inside each layer:

```
Persona.Api            controllers, middleware (correlation id), exception handler, DI
Persona.Application    use cases per feature: Authentication, Memories, Plans, Subscription, Entitlements, Outbox, Shared (idempotency, abstractions)
Persona.Domain         entities and value objects: User, Memory, OutboxMessage, Subscription, Plan, Entitlement
Persona.Infrastructure EF Core (PersonaDbContext, migrations, repositories, UnitOfWork), JWT, RabbitMQ, hosted services
```

Application depends only on Domain; ASP.NET and EF Core stay in Api/Infrastructure (e.g. `ICurrentUser`, `ICorrelationContext`).

## 3. Capture flow (implemented)

1. `POST /api/v1/memories` with `Idempotency-Key` (and optional `X-Correlation-Id`).
2. One transaction: reserve idempotency key → insert memory → insert outbox message → complete idempotency record → commit.
3. Respond `201 Created` with the memory (ADR-0008). Concurrent or repeated requests with the same key replay the stored response (ADR-0001).

Publishing is **not** part of capture. A memory is saved once the transaction commits; getting the message to RabbitMQ is the relay's job (PS-06). If publishing fails the memory is still saved and the outbox row stays pending.

## 4. Analysis flow

| Step | Status |
|---|---|
| Outbox relay publishes `MemoryAnalysisRequested` with confirms (PS-06) | Pending |
| AI consumes, extracts items, publishes `MemoryAnalysisCompleted` (`CausationId` = request `MessageId`) | Implemented |
| API consumes the result | Implemented (logs only) |
| Result stored in `memory_analyses`, status lifecycle, inbox dedupe (ADR-0003/0004/0006; PS-10, PS-11) | Pending |
| Failed-analysis message, dead-letter, retry limits, QoS (PS-58, PS-09) | Pending |

Message formats, topology and version policy are defined as versioned JSON Schemas in `contracts/`.

## 5. Operating rules

- Capture writes memory and outbox atomically; `201` means the memory is saved, not that it was analysed.
- Delivery is at-least-once; every consumer must tolerate duplicates (ADR-0006).
- Every query is filtered by `UserId`; the API adds that filter itself, never the LLM (ADR-0023).
- The AI service never writes the database; statuses are changed only by the API (ADR-0003).
- Memory text never appears in logs; logs carry `CorrelationId`, `CausationId`, `MessageId`, `MemoryId` (ADR-0018, PS-07).
- Every copy of memory text has a retention and deletion rule (ADR-0019).
- A new service, queue or database is a separate, documented decision (ADR).

## 6. Repository map

```
apps/          api · ai · mobile · rag (reserved)
contracts/     JSON Schemas + shared fixtures (source of truth for messages)
docs/          architecture · decisions (ADRs) · domain · development rules · mvp
tests/         api (xUnit + Testcontainers) · ai (pytest) · integration (Ollama, opt-in)
infrastructure/, workers/, scripts/   skeletons for deploy, extracted background services and dev scripts — see their READMEs
infrastructure/docker/   dev compose files (infra + apps); Dockerfiles live in apps/api and apps/ai
.github/       CI (api / ai / mobile jobs with path filters), issue and PR templates
```
