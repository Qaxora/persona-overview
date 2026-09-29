# ADR-0023: Query engine plans with the LLM and executes in .NET

- **Status:** Accepted
- **Date:** 2026-09-29
- **Decision code:** D-23

## Context

Users ask three kinds of questions:

- **Structured** — "How much did I spend on health in the last 3 months?" → filters + aggregation (SQL).
- **Semantic** — "What was that problem with my dog?" → similarity search (vectors / RAG).
- **Hybrid** — "Summarise my car problems since I bought it and their total cost" → both.

RAG is one retrieval strategy, not the query engine. Arithmetic by an LLM is unreliable. An LLM that writes SQL could read other users' data or run unintended statements.

## Decision

Like capture, a query is analysed first:

```
Question → LLM: query understanding → QueryPlan (validated JSON)
        → .NET executes the plan (SQL / vector / both)
        → results → LLM: answer synthesis with sources
```

Rules:

1. **The LLM never writes SQL.** It produces a `QueryPlan` — strategy (`sql` | `semantic` | `hybrid`), category ids (ADR-0022), time range, measure (`count` | `sum(money)` | ...), free-text search terms. The plan is validated like analysis output (ADR-0016). .NET builds parameterised queries and **always** adds the `UserId` filter itself.
2. **Numbers come from the database.** Sums, counts and averages are computed in SQL; the LLM only phrases the result.
3. **Answers cite their memories.** Every answer returns the memory ids it used (PS-23 "kaynaklı cevap").
4. **Degrade, don't fail.** If planning or synthesis times out (ADR-0011), return the structured results that are available.

## Consequences

- `QueryPlan` becomes a versioned contract (ADR-0021).
- Query understanding needs its own evaluation set, like capture analysis (PS-65).
- Event-time queries depend on resolved dates (PS-15) and derived projections (ADR-0004, ADR-0022).
