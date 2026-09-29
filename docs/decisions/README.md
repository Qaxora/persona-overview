# Selected decisions (ADR)

Persona records every significant design decision as a short Architecture Decision Record: the context, the options considered, the decision and its consequences. The full set (25 ADRs so far) lives in the private repository. These are the ones that best show how the system is shaped.

`PS-xx` and `#nn` identifiers refer to items in the private issue tracker. References to ADRs not included here point to the private set.

| ADR | Decision | Why it's here |
|---|---|---|
| [0001](0001-concurrent-idempotent-requests-replay.md) | Concurrent idempotent requests replay the first response | Safe client retries under concurrency: unique index + single transaction |
| [0005](0005-memory-correction-and-deletion.md) | Memory deletion is in the MVP; editing is not | Scope vs. trust vs. KVKK and store requirements |
| [0006](0006-duplicate-message-protection.md) | Duplicate message protection lives in the API inbox | At-least-once delivery handled boundary by boundary |
| [0016](0016-invalid-llm-items.md) | Invalid LLM items are dropped, valid ones are kept | How LLM output is validated before it touches data |
| [0020](0020-memory-content-is-special-category-data.md) | All memory content is special-category personal data | Privacy as a design constraint, not a feature |
| [0023](0023-query-engine-architecture.md) | Query engine plans with the LLM and executes in .NET | The LLM never writes SQL; numbers come from the database |
| [0024](0024-embedding-at-ingest.md) | Memory embeddings are computed at ingest, in the background | Where RAG fits, and what happens when the model changes |
