# ADR-0006: Duplicate message protection lives in the API inbox

- **Status:** Accepted
- **Date:** 2026-09-29
- **Decision code:** D-06

## Context

RabbitMQ guarantees **at-least-once** delivery: a message may arrive more than once. A memory travels through four boundaries:

```
Mobile ──HTTP──▶ API ──outbox──▶ RabbitMQ ──▶ AI ──▶ RabbitMQ ──▶ API
       (1)                (2)               (3)               (4)
```

| Boundary | How a duplicate happens | Protection today |
|---|---|---|
| (1) Mobile → API | Client retries after a timeout | `Idempotency-Key` (ADR-0001) |
| (2) API → broker | Outbox relay publishes, crashes before recording it, publishes again | none |
| (3) Broker → AI | AI processes, crashes before ack, message is redelivered | none |
| (4) Broker → API | Same completed message delivered twice | none |

`Idempotency-Key` only protects boundary (1). Boundaries (2)–(4) need the same idea keyed by `MessageId` (the inbox pattern). The AI service has no database.

## Options

1. **API inbox is the authority; AI keeps an in-process cache.** Rare duplicate LLM calls accepted.
2. **Give the AI service its own database for an inbox.** Exact once-only LLM calls; adds a datastore to operate.
3. **No deduplication; rely on idempotent writes only.** Simplest; duplicate LLM calls on every redelivery and no audit of processed messages.

## Decision

**Option 1.**

- **(4) API — authoritative.** An `inbox_messages` table stores the `MessageId` of every consumed message (unique). A message whose `MessageId` is already present is acknowledged and skipped. Insert into the inbox happens in the same transaction as the state change it causes. In addition, ADR-0004 matching (`CausationId == RequestMessageId`) ignores a result for a run that is already `Completed` or `Failed`.
- **(3) AI — best effort.** The consumer keeps recently processed `MessageId`s in memory with a short TTL and skips repeats. After a restart the cache is empty, so an occasional second LLM call is accepted; its result is deduplicated at (4).
- **(2) Relay.** No extra protection; a double publish is caught at (3) and (4).

## Consequences

- One migration adds `inbox_messages` (`MessageId` PK, `MessageType`, `ProcessedAtUtc`).
- A duplicate can cost one extra LLM call, never duplicate data.
- If LLM cost grows or duplicates become frequent, revisit Option 2.
- The inbox is also an audit of what the API has consumed, useful when tracing a `CorrelationId` (ADR-0002).
