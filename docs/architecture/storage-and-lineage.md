# Storage And Lineage

The platform separates durable metadata from large artifacts and transient processing state.

## Storage Boundaries

| Store | Owns | Does not own |
| --- | --- | --- |
| PostgreSQL | Document records, job definitions, ACLs, final job status, artifact pointers, parser versions, quality summaries, retention metadata. | Large parsed JSON blobs, page images, intermediate chunks, per-step logs, high-frequency progress events. |
| MinIO/S3 | Original files, derived safe files, normalized JSON, Markdown, text, table artifacts, thumbnails, page images, intermediate chunk artifacts, detailed lineage snapshots. | Hot mutable job state. |
| Redis/Queue backend | Active job state, locks, progress counters, worker heartbeats, short-lived retry metadata, in-flight lineage. | Durable lineage, final artifacts, ACL source of truth. |

## Why

Keeping every intermediate JSON blob and pipeline update in PostgreSQL can create write amplification and storage pressure. PostgreSQL should stay authoritative for metadata and final job state, while MinIO/S3 stores large immutable artifacts.

Document parsing creates many high-churn writes:

- page-level progress updates
- OCR block metadata
- chunk candidates
- parser debug traces
- intermediate normalized fragments
- retry and heartbeat updates

These should not hammer relational tables. Active state belongs in Redis with TTLs. Large artifacts belong in object storage. PostgreSQL should store compact pointers and final state transitions.

## State Model

| State category | Store | Example |
| --- | --- | --- |
| Final document record | PostgreSQL | document ID, tenant, source hash, status, ACL. |
| Final job record | PostgreSQL | job ID, final status, timestamps, parser version. |
| Artifact pointer | PostgreSQL | object key, content type, byte size, hash, retention class. |
| Active progress | Redis | current page, percent, worker ID, heartbeat. |
| Active retry metadata | Redis/queue backend | attempt count, next retry time, lock token. |
| Intermediate chunks | MinIO/S3 | page JSON, OCR blocks, chunk candidates. |
| Detailed lineage snapshot | MinIO/S3 | full parser config, step log, model versions. |
| Quality summary | PostgreSQL + MinIO/S3 | compact score in PostgreSQL, full report in object storage. |

## PostgreSQL Write Policy

PostgreSQL should receive low-frequency durable state changes:

- create document
- create parse job
- mark job accepted
- mark job running when worker starts
- mark job succeeded or failed
- register artifact pointers
- store compact quality summary
- store ACL and retention metadata

Avoid:

- updating progress every page
- storing large normalized document JSON directly in a row
- storing extraction logs in JSONB columns
- storing every chunk candidate in relational tables
- storing OCR block-level metadata for every page
- using PostgreSQL as a queue or hot state store

## Redis State Policy

Redis stores short-lived mutable state:

- progress counters
- current stage
- worker heartbeat
- lock tokens
- retry metadata
- recent routing decisions if needed for fast lookup

Redis keys should use TTLs so abandoned jobs do not leak memory.

Example:

```text
job:{job_id}:state
job:{job_id}:heartbeat
job:{job_id}:progress
job:{job_id}:retry
```

Redis state is operational state, not the audit source of truth.

## Object Storage Policy

Object storage keeps large immutable artifacts:

- original source file
- parser-safe derived file
- page-level extracted JSON
- OCR block JSON
- normalized document JSON
- canonical table HTML
- table metadata JSON
- chunk candidates
- full quality report
- full lineage report

Artifacts should be content-addressed or hash-verified so reprocessing is auditable.

## Lineage Fields

Every output artifact should be traceable to:

- source document ID
- source file hash
- artifact hash
- parser name and version
- worker image digest
- OCR engine and model version when applicable
- configuration version
- created timestamp
- job ID
- tenant or workspace ID when multi-tenant
- correlation ID

## Artifact Naming

Artifacts should use deterministic paths:

```text
tenants/{tenant_id}/inputs/{yyyy}/{mm}/{dd}/{source_hash}.{ext}
tenants/{tenant_id}/documents/{document_id}/derived/safe-input
tenants/{tenant_id}/documents/{document_id}/outputs/document.json
tenants/{tenant_id}/documents/{document_id}/outputs/document.md
tenants/{tenant_id}/documents/{document_id}/outputs/document.txt
tenants/{tenant_id}/documents/{document_id}/outputs/pages/{page_number}.json
tenants/{tenant_id}/documents/{document_id}/outputs/chunks/{chunk_id}.json
tenants/{tenant_id}/documents/{document_id}/outputs/tables/{table_id}.html
tenants/{tenant_id}/documents/{document_id}/outputs/tables/{table_id}.json
tenants/{tenant_id}/documents/{document_id}/metadata/quality.json
tenants/{tenant_id}/documents/{document_id}/metadata/lineage.json
tenants/{tenant_id}/documents/{document_id}/metadata/steps/{step_id}.json
```

The database stores pointers, hashes, sizes, content types, and retention metadata for these artifacts.

Source objects should be immutable. Reprocessing creates new derived artifacts and lineage records, not mutation of the original input.

## Lineage Persistence

Lineage has two layers:

| Layer | Store | Purpose |
| --- | --- | --- |
| Compact lineage summary | PostgreSQL | searchable metadata: parser name/version, worker image digest, source hash, final artifact hash. |
| Full lineage artifact | MinIO/S3 | detailed step-by-step pipeline trace, parser configs, model versions, warnings, and extraction statistics. |

This keeps auditability without forcing PostgreSQL to store huge JSON blobs.

## Recovery Behavior

If Redis state is lost:

- PostgreSQL still has the durable job record.
- Object storage still has completed artifacts.
- in-flight jobs can be marked stale and retried or reconciled.
- workers should use idempotent artifact writes and finalization steps.

If object storage artifact registration fails:

- do not mark the job succeeded in PostgreSQL.
- retry artifact registration or fail the job.

If PostgreSQL finalization fails:

- keep artifacts in a temporary/staged object key.
- retry finalization.
- garbage-collect abandoned staged artifacts after retention timeout.

## Anti-Patterns

- Store every chunk candidate in PostgreSQL.
- Store full extraction logs in PostgreSQL JSONB.
- Update job progress in PostgreSQL for every page.
- Treat Redis state as durable audit history.
- Store final artifact bytes in PostgreSQL.
- Let workers write unregistered artifacts without hashes and retention metadata.
