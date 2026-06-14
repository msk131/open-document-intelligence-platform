# Document Intelligence Architecture

```mermaid
flowchart TB
    USER["User / Batch Import / External System"]
    API["FastAPI Intake API"]
    VALIDATE["Validation\nsize, type, hash, corruption,\nimage magic bytes and dimensions"]
    STORE[("Object Storage\noriginal files and artifacts")]
    META[("PostgreSQL\njobs, metadata, quality")]
    HOT[("Redis\nactive state, locks,\nprogress, heartbeats")]
    ROUTEQUEUE["Queue Router\nworkload class selection"]
    QLIGHT["queue_light"]
    QOFFICE["queue_office"]
    QPDF["queue_pdf"]
    QOCR["queue_ocr"]
    QTABLE["queue_table"]
    QLLM["queue_llm"]
    WL["worker-light\nTXT, MD, HTML, CSV"]
    WO["worker-office\nDOCX, PPTX, XLSX, EML"]
    WP["worker-pdf-native\nborn-digital PDF"]
    WOC["worker-ocr\nscanned PDF, PNG, JPG, JPEG"]
    WT["worker-table\ntable repair"]
    WLLM["worker-llm\noptional enrichment"]

    DETECT["Pre-Flight Inspector\nMIME, magic bytes, page sampling,\ntext density, image density,\ncorruption, limits"]
    ROUTER["Parser Router\ncondition-based matrix"]
    PARSERS["Specialized Parser Engines\nDocling, Tika, PyMuPDF,\nPandoc, openpyxl"]
    OCR["OCR Engine\nTesseract / PaddleOCR\nscanned PDF, PNG, JPG, JPEG"]
    NORMALIZE["Normalizer\ncommon document model"]
    QUALITY["Quality Checks\nempty output, confidence,\nmissing pages, table issues"]
    PII["PII / Sensitive Data Gate\nredact or flag before indexing"]
    EXPORT["Artifact Export\nJSON, Markdown, text, tables"]
    CHUNK["Chunking\nsource anchors and context"]
    LLM["LLM Enrichment\nrate-limited optional stage"]
    DOWNSTREAM["Downstream Systems\nRAG, search, analytics"]
    OBS["Observability\nmetrics, traces, logs"]

    USER --> API
    API --> VALIDATE
    VALIDATE --> STORE
    VALIDATE --> META
    VALIDATE --> DETECT
    DETECT --> HOT
    DETECT --> ROUTER
    ROUTER --> ROUTEQUEUE
    ROUTEQUEUE --> QLIGHT
    ROUTEQUEUE --> QOFFICE
    ROUTEQUEUE --> QPDF
    ROUTEQUEUE --> QOCR
    ROUTEQUEUE --> QTABLE
    ROUTEQUEUE --> QLLM
    QLIGHT --> WL
    QOFFICE --> WO
    QPDF --> WP
    QOCR --> WOC
    QTABLE --> WT
    QLLM --> WLLM
    WL --> PARSERS
    WO --> PARSERS
    WP --> PARSERS
    WOC --> OCR
    WT --> PARSERS
    PARSERS --> NORMALIZE
    OCR --> NORMALIZE
    NORMALIZE --> QUALITY
    QUALITY --> PII
    PII --> EXPORT
    EXPORT --> CHUNK
    CHUNK --> LLM
    WLLM --> LLM
    EXPORT --> STORE
    EXPORT --> META
    CHUNK --> STORE
    LLM --> STORE
    LLM --> META
    EXPORT --> DOWNSTREAM
    API --> OBS
    WL --> OBS
    WO --> OBS
    WP --> OBS
    WOC --> OBS
    WT --> OBS
    WLLM --> OBS
    CHUNK --> OBS
    LLM --> OBS
    PII --> OBS
    QUALITY --> OBS
```

## Flow Summary

1. File is uploaded or imported.
2. Platform validates size, type, hash, basic file health, and image safety checks for PNG/JPEG inputs.
3. Original file is stored and pre-flight inspection samples the document.
4. Router chooses a workload class from document condition, not extension alone.
5. Queue router sends work to light, office, PDF, OCR, or table queues.
6. Specialized workers run only the parser dependencies they need.
7. Output is normalized into a common document model.
8. Quality checks flag incomplete or low-confidence extraction.
9. PII/sensitive-data gates redact or flag content before open search/RAG indexing.
10. Artifacts are exported with source anchors for RAG, search, analytics, and review.
11. Optional LLM enrichment runs through a separate rate-limited queue.

## Compute Tiering

```mermaid
flowchart TB
    GATEWAY["Enterprise API Gateway"]
    PROFILE["Fast Profiling Stage"]
    CPU["Tier-1 CPU Workers\nPyMuPDF, python-docx, openpyxl\nhigh replica count"]
    GPU["Tier-2 GPU/VLM Workers\nDocling, PaddleOCR, visual layout\nscale-to-zero / spot GPU"]
    NORM["Document Normalizer v1"]

    GATEWAY --> PROFILE
    PROFILE -->|"text/direct stream"| CPU
    PROFILE -->|"visual/VLM matrix"| GPU
    CPU --> NORM
    GPU --> NORM
```

## Storage Ownership

```mermaid
flowchart TB
    WORKER["Parser workers"]
    PG[("PostgreSQL\nmetadata, ACLs, job definitions,\nfinal status, artifact pointers")]
    REDIS[("Redis\nactive progress, locks,\nheartbeats, retry state")]
    S3[("MinIO/S3\noriginal files, outputs,\nchunks, full lineage")]

    WORKER -->|"final status + pointers"| PG
    WORKER -->|"hot mutable state"| REDIS
    WORKER -->|"large immutable artifacts"| S3
    S3 -->|"object key + hash"| PG
```

## Quality And Security Gate

```mermaid
flowchart LR
    NORM["normalized output"]
    QA["quality assertion\nVERIFIED / REVIEW_REQUIRED / FAIL"]
    REDACT["PII redaction or flagging"]
    REVIEW["human review queue"]
    INDEX["OpenSearch / RAG index"]

    NORM --> QA
    QA -->|"FAIL"| REVIEW
    QA -->|"REVIEW_REQUIRED"| REVIEW
    QA -->|"VERIFIED"| REDACT
    REDACT --> INDEX
```

## Worker Isolation

```mermaid
flowchart LR
    API["api\nno heavy parser deps"]
    QR["queue router"]
    L["worker-light\nsmall image"]
    O["worker-office\noffice deps"]
    P["worker-pdf-native\nPDF/layout deps"]
    OCRW["worker-ocr\nOCR/ML deps"]
    T["worker-table\ntable repair deps"]
    LLM["worker-llm\nrate-limited LLM deps"]

    API --> QR
    QR --> L
    QR --> O
    QR --> P
    QR --> OCRW
    QR --> T
    QR --> LLM
```

## Queue Isolation

```mermaid
flowchart LR
    PREFLIGHT["Preflight classification"]
    LIGHT["queue_light\nshort timeout\nhigh concurrency"]
    HEAVY["queue_ocr / queue_table\nlong timeout\nlow concurrency"]
    LLMQ["queue_llm\nrate-limited\nbudgeted retries"]
    TXT["1 KB TXT"]
    PDF["500-page scanned PDF"]

    TXT --> PREFLIGHT
    PDF --> PREFLIGHT
    PREFLIGHT --> LIGHT
    PREFLIGHT --> HEAVY
    PREFLIGHT --> LLMQ
```

## LLM Isolation

```mermaid
flowchart LR
    PARSE["Deterministic parsing complete"]
    CHUNKS["source-anchored chunks"]
    QLLM["queue_llm\nrate limits, token budgets"]
    WLLM["worker-llm\ncircuit breaker, idempotency"]
    ART["LLM output artifacts\nhash + lineage"]

    PARSE --> CHUNKS
    CHUNKS --> QLLM
    QLLM --> WLLM
    WLLM --> ART
```

## Idempotent Pipeline Trace

```mermaid
flowchart LR
    STAGE["pipeline stage"]
    KEY["idempotency key\njob + stage + input hash + config"]
    TRACE["OpenTelemetry trace context"]
    STAGED["staged artifact"]
    FINAL["final artifact pointer"]

    STAGE --> KEY
    STAGE --> TRACE
    KEY --> STAGED
    TRACE --> STAGED
    STAGED --> FINAL
```

## PDF Preflight Routing

```mermaid
flowchart LR
    PDF["PDF upload"]
    PROBE["Bounded page sampling\nfirst, middle, last"]
    TEXT["Text density check"]
    IMAGE["Image density check"]
    NATIVE["queue_pdf\nworker-pdf-native"]
    OCRQ["queue_ocr\nworker-ocr"]
    REJECT["reject / quarantine"]

    PDF --> PROBE
    PROBE --> TEXT
    PROBE --> IMAGE
    TEXT -->|"selectable text found"| NATIVE
    IMAGE -->|"image-dominant or near-zero text"| OCRQ
    PROBE -->|"corrupt, encrypted, over limit"| REJECT
```

## Image Routing

```mermaid
flowchart LR
    IMG["PNG / JPG / JPEG upload"]
    SNIFF["MIME + magic-byte sniffing"]
    SAFE["Image safety checks\nsize, max pixels, corruption,\nEXIF orientation"]
    OCR["OCR strategy\nTesseract / PaddleOCR"]
    MODEL["Normalized document model\ntext blocks, bounding boxes,\nconfidence, dimensions"]
    QUALITY["Quality checks\nempty text, low confidence,\nunsupported language"]

    IMG --> SNIFF
    SNIFF --> SAFE
    SAFE --> OCR
    OCR --> MODEL
    MODEL --> QUALITY
```

## Table Normalization

```mermaid
flowchart LR
    EXCEL["Excel/openpyxl\nrows, columns, merged cells"]
    PDFT["PDF/Docling\nbounding boxes, inferred cells"]
    OCRT["OCR/PaddleOCR\ntext regions, confidence"]
    OFFICE["DOCX/PPTX\nnative table nodes"]
    ADAPT["Parser-specific adapters"]
    HTML["canonical sanitized\nHTML table"]
    META["table metadata\nlineage, confidence, warnings"]
    JSON["derived JSON export"]
    MD["Markdown export\nonly when non-lossy"]

    EXCEL --> ADAPT
    PDFT --> ADAPT
    OCRT --> ADAPT
    OFFICE --> ADAPT
    ADAPT --> HTML
    ADAPT --> META
    HTML --> JSON
    META --> JSON
    HTML --> MD
```
