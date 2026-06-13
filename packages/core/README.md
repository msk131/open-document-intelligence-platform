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
