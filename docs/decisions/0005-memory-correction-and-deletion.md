# ADR-0005: Memory deletion is in the MVP; editing is not

- **Status:** Accepted
- **Date:** 2026-09-29
- **Decision code:** D-09

## Context

A user may want to change or remove a memory after writing it. The MVP goal is to collect as much genuine memory data as possible, which argues for a small feature set. At the same time:

- Apple (App Store Review Guideline 5.1.1(v)) and Google Play require in-app account deletion for apps that allow account creation.
- KVKK (articles 7 and 11) gives the user the right to have personal data erased. Persona stores the most personal category of data there is.
- The product is positioned as privacy-focused. A user who cannot remove something written by mistake stops trusting the product, which reduces data collection rather than increasing it.

## Options

1. **No edit, no delete in the MVP.** Smallest scope; blocks store approval and conflicts with KVKK.
2. **Delete in the MVP, edit later.** Trust and compliance covered; correction semantics deferred.
3. **Edit and delete in the MVP.** Largest scope.

## Decision

**Option 2.**

- **Memory deletion (PS-19) is in the MVP** as a soft delete: the memory is hidden from the user immediately; permanent removal follows after a waiting period.
- **Account deletion (PS-32) is in the MVP**, with the waiting period defined in PS-32 (proposed 30 days) and a permanent-deletion job.
- Permanent deletion removes the memory **and all its derived data** — every `memory_analyses` row (ADR-0004), embeddings, and any later projections.
- An analysis result that arrives for a soft-deleted memory is ignored and logged.
- **Memory editing is post-MVP.** When it is added it must follow this rule: the previous text is kept as an append-only revision, the memory shows the latest text, and the new text gets a new analysis run while older runs become `IsCurrent = false` (ADR-0004). Each edit starts a new workflow / `CorrelationId` (ADR-0002).

## Consequences

- PS-19 and PS-32 stay in MVP scope; PS-18 (correction) moves after the MVP.
- ADR-0004's "the system never deletes analyses" holds; the only deletion path is the user's erasure right.
- The data-export feature (PS-33) must respect soft-deleted memories.
- Store submission (PS-45) and the privacy text (PS-39) can state that users can delete memories and their account in-app.
