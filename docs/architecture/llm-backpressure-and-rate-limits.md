# LLM Backpressure And Rate Limits

LLM inference is an optional enrichment path. It should not block deterministic parsing, OCR, table repair, or artifact export.

## Design Rule

All LLM work goes through `queue_llm` and `worker-llm`.

The parse pipeline should be able to complete without LLM enrichment when LLM providers are unavailable, rate-limited, or disabled by deployment policy.

## Controls

| Control | Purpose |
| --- | --- |
| Per-tenant request budget | Prevent one tenant from consuming all LLM capacity. |
| Per-tenant token budget | Control cost and provider quota usage. |
| Provider rate limiter | Respect external or local model throughput limits. |
| Circuit breaker | Stop retry storms when provider failures spike. |
| Bounded retry | Retry transient timeout/rate-limit failures safely. |
| Dead-letter queue | Preserve failed enrichment jobs for review/replay. |
| Idempotency key | Prevent duplicate writes on retry. |
| Prompt/config version | Make LLM outputs reproducible and auditable. |

## Retry Policy

Retry only when:

- the request is idempotent
- the same input artifact hash is used
- the same prompt/config version is used
- the provider error is transient
- the tenant still has budget

Do not retry when:

- token budget is exceeded
- content policy rejects the request
- input artifact changed
- prompt/config version changed mid-stage
- repeated provider failures trip the circuit breaker

## Metadata For Every LLM Call

Every LLM output artifact should record:

- trace ID
- job ID
- stage ID
- idempotency key
- tenant/workspace ID
- model/provider
- prompt version
- config version
- input artifact key
- input artifact hash
- output artifact key
- output artifact hash
- token count
- latency
- retry count
- policy decisions

## Anti-Patterns

- Running LLM calls inside OCR or table workers.
- Letting LLM jobs share `queue_ocr` or `queue_table`.
- Retrying provider failures without a budget.
- Treating LLM enrichment as required for base parsing success.
- Storing prompts and outputs only in PostgreSQL rows.
