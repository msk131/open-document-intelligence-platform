# ADR 0006: Enterprise Compute And Security

## Status

Accepted.

## Context

Enterprise workloads are mixed. A single gateway may receive thousands of clean text emails while another user uploads a large scanned document requiring OCR, layout detection, or VLM processing. Enterprises also require tenant isolation, lineage, quality gates, and deployment inside controlled boundaries.

## Decision

Use explicit compute tiers and enterprise guardrails:

- Tier-1 CPU workers for fast deterministic parsing.
- Tier-2 GPU/VLM workers for scanned/visual workloads.
- Tier-3 enrichment workers for table repair and optional LLM tasks.
- tenant-scoped immutable object storage paths.
- quality assertion profiles before downstream indexing.
- PII redaction or sensitive-data flagging before OpenSearch/RAG.
- on-premises, VPC-isolated, and air-gapped deployment support.

## Consequences

Benefits:

- Better cost control for expensive GPU workloads.
- Clean text workloads are not blocked by scanned documents.
- Enterprise audit and lineage requirements are explicit.
- Sensitive content is controlled before indexing.
- Deployment can satisfy private-boundary requirements.

Costs:

- More deployment profiles.
- More queue and autoscaling policies.
- More quality/security metadata.
- More testing around tenant isolation and redaction idempotency.
