# Document Intelligence Architecture

```mermaid
flowchart TB
    USER["User / Batch Import / External System"]
    API["FastAPI Intake API"]
    VALIDATE["Validation\nsize, type, hash, corruption"]
    STORE[("Object Storage\noriginal files and artifacts")]
    META[("PostgreSQL\njobs, metadata, quality")]
    QUEUE["Job Queue\nRedis / Kafka"]
    WORKER["Parser Worker"]

    DETECT["File Detection\nMIME, extension, encrypted,\nscanned vs digital"]
    ROUTER["Parser Router"]
    PARSERS["Open-Source Parsers\nDocling, Tika, Unstructured,\nPyMuPDF, Pandoc, openpyxl"]
    OCR["OCR Engine\nTesseract / PaddleOCR"]
    NORMALIZE["Normalizer\ncommon document model"]
    QUALITY["Quality Checks\nempty output, confidence,\nmissing pages, table issues"]
    EXPORT["Artifact Export\nJSON, Markdown, text, tables"]
    DOWNSTREAM["Downstream Systems\nRAG, search, analytics"]
    OBS["Observability\nmetrics, traces, logs"]

    USER --> API
    API --> VALIDATE
    VALIDATE --> STORE
    VALIDATE --> META
    VALIDATE --> QUEUE
    QUEUE --> WORKER
    WORKER --> DETECT
    DETECT --> ROUTER
    ROUTER --> PARSERS
    ROUTER --> OCR
    PARSERS --> NORMALIZE
    OCR --> NORMALIZE
    NORMALIZE --> QUALITY
    QUALITY --> EXPORT
    EXPORT --> STORE
    EXPORT --> META
    EXPORT --> DOWNSTREAM
    API --> OBS
    WORKER --> OBS
    QUALITY --> OBS
```

## Flow Summary

1. File is uploaded or imported.
2. Platform validates size, type, hash, and basic file health.
3. Original file is stored and parse job is queued.
4. Worker detects file type and document condition.
5. Parser router chooses the best open-source parser or OCR path.
6. Output is normalized into a common document model.
7. Quality checks flag incomplete or low-confidence extraction.
8. Artifacts are exported for RAG, search, analytics, and review.

