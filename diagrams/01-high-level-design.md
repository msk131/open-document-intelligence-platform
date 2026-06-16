# High-Level Design

```mermaid
flowchart TB
    USER["User / Batch Import"]
    API["Intake API"]
    INSPECT["Preflight Inspector\nMIME, magic bytes, size,\ntext density, corruption"]
    STORE[("Object Storage\nsource files and artifacts")]
    META[("PostgreSQL\njobs, metadata, lineage")]
    ROUTER["Parser Router"]
    QUEUES["Workload Queues\nlight, office, pdf, ocr, table, llm"]
    WORKERS["Specialized Workers\nparser / OCR / table / LLM"]
    NORMALIZE["Normalizer\ncommon document model"]
    QUALITY["Quality Gate\nconfidence, completeness, review"]
    INDEX["Search / RAG Index"]
    OBS["Observability"]

    USER --> API
    API --> STORE
    API --> META
    API --> INSPECT
    INSPECT --> ROUTER
    ROUTER --> QUEUES
    QUEUES --> WORKERS
    WORKERS --> NORMALIZE
    NORMALIZE --> QUALITY
    QUALITY --> STORE
    QUALITY --> META
    QUALITY --> INDEX
    API --> OBS
    WORKERS --> OBS
    QUALITY --> OBS
```

## Notes

- Route by document condition, not file extension alone.
- Keep OCR, table, and LLM work isolated from light parsing.
- Store large artifacts in object storage and durable metadata in PostgreSQL.
- Index only verified or policy-approved output.
