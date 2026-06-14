# Storage Package

Shared storage abstractions.

Owns:

- artifact pointer contracts
- object key conventions
- source and output artifact metadata
- content hash helpers
- staged artifact registration contracts
- retention metadata contracts

This package should not store large parsed blobs in PostgreSQL directly.

## Storage Boundary

PostgreSQL stores compact metadata and pointers:

- document ID
- job ID
- artifact key
- content type
- byte size
- hash
- retention class
- final status

MinIO/S3 stores artifact payloads:

- original files
- derived safe files
- normalized JSON
- page/chunk JSON
- canonical table HTML
- full quality report
- full lineage report

Redis stores hot mutable state and should not be treated as durable audit history.

## Tenant Object Keys

Source objects should use immutable tenant-scoped paths:

```text
tenants/{tenant_id}/inputs/{yyyy}/{mm}/{dd}/{source_hash}.{ext}
```

Derived artifacts should live under:

```text
tenants/{tenant_id}/documents/{document_id}/...
```

The database stores object keys, hashes, sizes, content types, tenant ID, retention class, and encryption metadata.
