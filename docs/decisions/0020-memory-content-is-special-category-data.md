# ADR-0020: All memory content is treated as special-category personal data

- **Status:** Accepted
- **Date:** 2026-09-29
- **Decision code:** D-20

## Context

KVKK article 6 defines special categories of personal data (health, religion, political opinion, sexual life, criminal records, biometric data, ...) with stricter rules: explicit consent and additional safeguards. Memories routinely contain such data ("dişçiye gittim", "kan tahlilim çıktı"). Persona cannot know in advance which memory does.

## Decision

All memory content and everything derived from it (analyses, embeddings) is handled as special-category data:

- explicit consent at sign-up covers processing of such data (PS-39);
- database storage and backups are encrypted at rest (PS-36, PS-37);
- transport is TLS everywhere, including between services;
- transfer of memory text to third parties, especially abroad, follows the strictest path (ADR-0015); options that avoid it (self-hosted models, on-device analysis) are preferred where quality allows;
- access to production data is limited and logged.

## Consequences

- Hosting and provider choices (ADR-0013) must support encryption at rest and a suitable region.
- Field-level encryption of memory text can be evaluated later; disk/database encryption is the minimum.
