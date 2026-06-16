# Low-Level Design

```mermaid
sequenceDiagram
    participant Client
    participant API as Intake API
    participant Store as Object Storage
    participant Meta as Metadata DB
    participant Inspect as Preflight Inspector
    participant Router as Parser Router
    participant Queue as Workload Queue
    participant Worker as Parser Worker
    participant Norm as Normalizer
    participant Quality as Quality Gate
    participant Index as Search/RAG Index

    Client->>API: Upload file
    API->>Store: Store original file
    API->>Meta: Create parsing job
    API->>Inspect: Inspect MIME, magic bytes, pages, density
    Inspect->>Router: Return document condition
    Router->>Queue: Select workload queue
    Queue->>Worker: Dispatch parsing job
    Worker->>Store: Read source file
    Worker->>Norm: Emit normalized blocks, tables, metadata
    Norm->>Quality: Validate confidence and completeness
    Quality->>Store: Persist artifacts and lineage
    Quality->>Meta: Update status and quality profile
    Quality->>Index: Publish verified chunks
```

## Key Controls

| Control | Purpose |
| --- | --- |
| Preflight inspection | Prevent wrong parser selection and unsafe files. |
| Workload queues | Isolate fast text jobs from OCR/table-heavy jobs. |
| Normalized model | Give downstream systems one stable output shape. |
| Quality gate | Stop empty or low-confidence extraction from being indexed silently. |
| Lineage metadata | Preserve source hash, parser version, page anchors, and confidence. |
