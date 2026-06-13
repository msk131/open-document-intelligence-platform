# ADR 0004: Storage And Lineage Boundaries

## Status

Accepted.

## Context

Document parsing pipelines produce a large amount of intermediate state:

- page-level progress
- OCR blocks
- layout fragments
- chunk candidates
- parser debug logs
- table repair metadata
- detailed lineage traces

Storing all of this directly in PostgreSQL would create heavy write amplification, large JSONB rows, indexing pressure, backup bloat, and degraded query performance.

## Decision

Use storage by ownership:

| Store | Responsibility |
| --- | --- |
| PostgreSQL | final document metadata, ACLs, job definitions, final status, compact quality summaries, artifact pointers |
| Redis | active mutable pipeline state, locks, progress, heartbeats, retry metadata |
| MinIO/S3 | original files, derived files, normalized outputs, intermediate chunks, full lineage and quality artifacts |

PostgreSQL stores pointers to artifacts, not the artifact payloads.

## Consequences

Benefits:

- Reduces PostgreSQL write pressure.
- Keeps large blobs out of relational rows.
- Improves backup and query performance.
- Allows object storage lifecycle policies for large artifacts.
- Preserves detailed lineage without bloating relational tables.
- Allows Redis state loss to be recovered from durable job records and artifacts.

Costs:

- More storage integration code.
- Need artifact registration and garbage collection.
- Need reconciliation jobs for staged or orphaned artifacts.
- Need signed URLs or controlled artifact access.

This tradeoff is accepted because document intelligence pipelines are artifact-heavy and high-churn.
