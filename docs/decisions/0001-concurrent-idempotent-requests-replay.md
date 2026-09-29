# ADR-0001: Concurrent idempotent requests replay the first response

- **Status:** Accepted
- **Date:** 2026-09-29 (decided 2026-09-27, implemented in #130)
- **Decision code:** D-01

## Context

`POST /api/v1/memories` requires an `Idempotency-Key`. Two requests with the same key and the same body can arrive at the same time — typically a mobile client retrying after a timeout.

Idempotency reservation, the memory, and the outbox message are written in one transaction. The `InProgress` reservation of the first request is therefore invisible to the second one until commit. The second request waits on the unique index `IX_idempotency_records_UserId_Key_Operation` and then receives PostgreSQL `23505`. Before PS-02 this surfaced as `500`.

## Options

1. **Replay** — the second request returns the first request's stored response (`201` + same body).
2. **409 Conflict** — tell the client another request is in progress.

In this API `409` already means "same key, different request body". Returning it for an identical request would blur that meaning, and because of the single transaction the "in progress" state cannot actually be observed.

## Decision

- Same key + same body, concurrent or sequential → **replay** the stored response (`201`, identical body).
- Same key + different body → **`409 Conflict`** (`IdempotencyConflictException`).
- Only the unique violation on the idempotency index is treated as a reservation conflict; any other `DbUpdateException` stays a `500`.

Implementation: `UnitOfWork.SaveChangesAsync` maps the `23505` on that index to `IdempotencyReservationConflictException`; `MemoryService` rolls back, clears the change tracker, re-reads the record and replays it.

## Consequences

- Clients can retry safely; a retry never shows an error for a request that already succeeded.
- `IdempotencyInProgressException` remains as a defensive path but is not expected with the single-transaction design.
- If the stored record disappears between the conflict and the re-read (future cleanup job, PS-69), `MemoryService` currently dereferences `null`; guard it when PS-69 lands.
- Covered by integration tests in `MemoryCaptureTests` (concurrent replay, different-body conflict, rollback).
