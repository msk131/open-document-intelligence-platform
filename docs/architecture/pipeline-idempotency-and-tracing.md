# Pipeline Idempotency And Tracing

The platform uses asynchronous multi-stage processing. Every stage must be traceable and retry-safe.

## Pipeline Stages

Typical stages:

```text
intake
  -> preflight
  -> parse / OCR
  -> normalize
  -> table repair
  -> chunk
  -> optional LLM enrichment
  -> quality
  -> export
  -> finalize
```

## Trace Propagation

Every stage should propagate:

- OpenTelemetry trace ID
- job ID
- document ID
- stage ID
- correlation ID
- tenant/workspace ID
- input artifact hash
- output artifact hash
- worker name and version

Queue messages must carry trace context so spans connect across API, queue, worker, storage, and export.

## Idempotency Contract

Each stage must have an idempotency key:

```text
{job_id}:{stage_name}:{input_artifact_hash}:{stage_config_version}
```

Stage retries should:

- read the same input artifact
- write to deterministic staged artifact paths
- verify output hashes
- finalize once
- avoid duplicate PostgreSQL finalization rows
- avoid duplicate PII redaction outputs

## Artifact Finalization

Workers should write artifacts in two phases:

```text
staged artifact write
  -> hash verification
  -> artifact pointer registration
  -> final status update
```

If a worker crashes after staged write but before finalization, reconciliation can either finalize the staged artifact or garbage-collect it after timeout.

## PII Redaction Idempotency

Redaction must be deterministic for the same input artifact and config version.

Record:

- redaction policy version
- detector/model version
- input artifact hash
- redacted output hash
- decision summary

Retries must not apply redaction repeatedly to an already redacted artifact unless explicitly configured as a new stage.

## Anti-Patterns

- Retrying a stage with a different input artifact under the same stage ID.
- Writing final artifacts directly without staged paths.
- Updating PostgreSQL before artifact hash verification.
- Losing trace context at queue boundaries.
- Treating PII redaction as a non-idempotent side effect.
