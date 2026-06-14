# Observability Package

Shared observability helpers.

Owns:

- correlation ID propagation
- structured log field names
- metric names
- tracing conventions
- queue and worker telemetry contracts

## Trace Contract

Every queue message and worker stage should propagate:

- OpenTelemetry trace ID
- correlation ID
- job ID
- document ID
- stage ID
- tenant/workspace ID
- input artifact hash
- output artifact hash when available
- worker name and version

The trace must cross API, preflight, queue router, workers, storage, optional LLM enrichment, and final export.
