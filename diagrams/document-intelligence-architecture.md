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
    WL["worker-light\nTXT, MD, HTML, CSV"]
    WO["worker-office\nDOCX, PPTX, XLSX, EML"]
    WP["worker-pdf-native\nborn-digital PDF"]
    WOC["worker-ocr\nscanned PDF, PNG, JPG, JPEG"]
    WT["worker-table\ntable repair"]

    DETECT["Pre-Flight Inspector\nMIME, magic bytes, page sampling,\ntext density, image density,\ncorruption, limits"]
    ROUTER["Parser Router\ncondition-based matrix"]
    PARSERS["Specialized Parser Engines\nDocling, Tika, PyMuPDF,\nPandoc, openpyxl"]
    OCR["OCR Engine\nTesseract / PaddleOCR\nscanned PDF, PNG, JPG, JPEG"]
    NORMALIZE["Normalizer\ncommon document model"]
    QUALITY["Quality Checks\nempty output, confidence,\nmissing pages, table issues"]
    EXPORT["Artifact Export\nJSON, Markdown, text, tables"]
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
    QLIGHT --> WL
    QOFFICE --> WO
    QPDF --> WP
    QOCR --> WOC
    QTABLE --> WT
    WL --> PARSERS
    WO --> PARSERS
    WP --> PARSERS
    WOC --> OCR
    WT --> PARSERS
    PARSERS --> NORMALIZE
    OCR --> NORMALIZE
    NORMALIZE --> QUALITY
    QUALITY --> EXPORT
    EXPORT --> STORE
    EXPORT --> META
    EXPORT --> DOWNSTREAM
    API --> OBS
    WL --> OBS
    WO --> OBS
    WP --> OBS
    WOC --> OBS
    WT --> OBS
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
9. Artifacts are exported for RAG, search, analytics, and review.

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

    API --> QR
    QR --> L
    QR --> O
    QR --> P
    QR --> OCRW
    QR --> T
```

## Queue Isolation

```mermaid
flowchart LR
    PREFLIGHT["Preflight classification"]
    LIGHT["queue_light\nshort timeout\nhigh concurrency"]
    HEAVY["queue_ocr / queue_table\nlong timeout\nlow concurrency"]
    TXT["1 KB TXT"]
    PDF["500-page scanned PDF"]

    TXT --> PREFLIGHT
    PDF --> PREFLIGHT
    PREFLIGHT --> LIGHT
    PREFLIGHT --> HEAVY
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
