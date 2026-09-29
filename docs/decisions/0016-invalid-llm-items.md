# ADR-0016: Invalid LLM items are dropped, valid ones are kept

- **Status:** Accepted
- **Date:** 2026-09-29
- **Decision code:** D-16

## Context

`ExtractionResult.model_validate` accepts or rejects the whole LLM response. In an integration run, one item with `temporal: {"expression": null}` caused the entire analysis to fail and the message to be dropped silently, losing the valid items too. The project prefers availability of captured information over consistency of derived intelligence.

## Options

1. **Reject the whole analysis** on any invalid item.
2. **Drop invalid items, keep valid ones.** If no valid item remains, the run is `Failed`.
3. **Repair heuristically** (e.g. turn an empty `temporal` object into `null`). Hides model behaviour; risky if it grows.

## Decision

**Option 2.** Items are validated one by one. Invalid items are dropped and logged with the reason and the run's `CorrelationId`; valid items are kept. A run whose items were **all** dropped as invalid (or an unparsable response) ends as `Failed` and is reported with a failed-analysis message (PS-58), never silently dropped. A response that legitimately contains no items — e.g. "Merhaba Persona, nasılsın?" — is `Completed` with an empty list, not a failure. Schema-constrained generation (structured outputs, #135) is the first line of defence; this rule is the second.

## Consequences

- Partial analyses are possible; the count of dropped items should be recorded for evaluation (PS-65).
- Permanent failures go to the dead-letter path (PS-09) instead of disappearing.
