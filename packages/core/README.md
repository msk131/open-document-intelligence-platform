# Core Package

Shared contracts that every service can use without pulling parser dependencies.

Owns:

- document IDs
- job states
- error codes
- routing enums
- artifact metadata contracts

## State Ownership

Core job states should distinguish durable status from transient progress.

Durable status belongs in PostgreSQL:

- accepted
- running
- review_required
- succeeded
- failed
- quarantined

Transient progress belongs in Redis:

- current page
- current stage
- heartbeat
- percent complete
- retry countdown

Large lineage and output payloads belong in MinIO/S3, referenced by artifact metadata contracts.

## Idempotency Contract

Every retryable stage should derive an idempotency key from:

```text
job_id + stage_name + input_artifact_hash + stage_config_version
```

Stages should write to deterministic staged artifact paths, verify hashes, then finalize once. This prevents duplicate database writes, duplicate artifacts, and repeated redaction side effects when workers retry after a crash.

## Tenant And Policy Contracts

Shared contracts should carry:

- tenant ID
- workspace ID when applicable
- retention class
- quality status
- PII/redaction status
- indexing approval status

Downstream indexing should require quality and sensitive-data gates to pass policy.
