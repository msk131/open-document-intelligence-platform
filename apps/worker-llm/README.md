# Worker LLM

Handles optional LLM-based enrichment after deterministic parsing has completed.

Target workloads:

- summarization
- semantic extraction
- entity normalization
- redaction review
- classification
- RAG-oriented enrichment

Runtime profile:

- Slow path.
- Low concurrency.
- Rate-limit aware.
- Token-budget controlled.
- Provider circuit breakers.
- Strict timeout and retry policy.

Responsibilities:

- Consume only `queue_llm`.
- Enforce per-tenant and per-provider rate limits.
- Use idempotency keys for every request.
- Store prompts, model names, config versions, input hashes, and output hashes in lineage.
- Write outputs as staged artifacts before final registration.
- Never block OCR, parsing, table repair, or light text queues.
