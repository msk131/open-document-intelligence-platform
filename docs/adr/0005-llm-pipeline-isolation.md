# ADR 0005: LLM Pipeline Isolation

## Status

Accepted.

## Context

LLM inference has different operational behavior from OCR and deterministic parsing:

- slower latency
- provider rate limits
- token budgets
- timeout variability
- policy failures
- higher cost

If LLM work shares queues or workers with parsing, it can block core document extraction.

## Decision

LLM enrichment is optional and isolated behind `queue_llm` and `worker-llm`.

LLM stages must use:

- per-tenant request and token budgets
- provider-specific rate limiting
- bounded timeout retries
- circuit breakers
- idempotency keys
- prompt/config versioning
- input and output artifact hashes
- OpenTelemetry trace propagation

## Consequences

Benefits:

- Base parsing can succeed even when LLM providers are slow or unavailable.
- OCR and table repair are not blocked by LLM latency.
- Cost and quota controls are explicit.
- LLM outputs are auditable and replayable.

Costs:

- Additional worker and queue.
- More policy configuration.
- More lineage metadata.
- More test cases for retry/idempotency behavior.
