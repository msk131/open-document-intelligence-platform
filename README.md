# Open Document Intelligence Platform

This project designs and builds an open-source document parsing and intelligence platform for many file types: PDF, DOCX, PPTX, XLSX, CSV, HTML, Markdown, TXT, emails, PNG, JPEG/JPG images, and scanned documents.

The goal is to convert messy enterprise documents into reliable structured output that can be searched, analyzed, audited, indexed for RAG, or processed by downstream automation.

## Why This Project Matters

Enterprise document processing is usually messy. Different teams store knowledge in different formats, scanned files lose structure, tables are hard to extract, and naive text extraction destroys layout and context.

This platform focuses on production-grade document understanding:

- File-type detection.
- Parser routing.
- OCR fallback.
- Layout extraction.
- Table extraction.
- Metadata extraction.
- Quality scoring.
- Normalized document model.
- Chunking for search and RAG.
- Repeatable parsing jobs with lineage.

## What We Are Building

The first version will parse multiple document formats into a normalized representation with text blocks, headings, tables, metadata, pages, source locations, and extraction confidence.

Primary capabilities:

| Capability | Purpose |
| --- | --- |
| File intake | Accept documents through API, batch folder, or object storage. |
| Type detection | Detect MIME type, extension mismatch, encryption, corruption, and size limits. |
| Parser routing | Choose the best parser by file type and document condition. |
| OCR | Extract text from scanned PDFs and images. |
| Layout extraction | Preserve pages, sections, headings, paragraphs, tables, and reading order. |
| Table extraction | Convert tables into structured rows, cells, and confidence scores. |
| Metadata extraction | Capture author, timestamps, language, page count, hash, and parser version. |
| Normalized output | Emit JSON/Markdown/text artifacts for downstream systems. |
| Quality checks | Detect empty extraction, low OCR confidence, missing pages, and table failures. |

## Supported File Types

| Family | Extensions | MIME types | Primary path | Notes |
| --- | --- | --- | --- | --- |
| PDF | `.pdf` | `application/pdf` | PDF parser, OCR fallback | Detect scanned vs born-digital pages. |
| Word | `.docx` | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | Office parser | Preserve headings, paragraphs, tables, and metadata. |
| PowerPoint | `.pptx` | `application/vnd.openxmlformats-officedocument.presentationml.presentation` | Office parser | Extract slide text, notes, and reading order. |
| Excel | `.xlsx` | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | Spreadsheet parser | Export sheets and tables as structured artifacts. |
| CSV | `.csv` | `text/csv`, `application/csv` | CSV parser | Protect exports from CSV formula injection. |
| Web/text | `.html`, `.htm`, `.md`, `.txt` | `text/html`, `text/markdown`, `text/plain` | Text/Pandoc parser | Normalize headings, links, and plain text. |
| Email | `.eml` | `message/rfc822` | Email parser | Extract headers, body, and attachments. |
| Images | `.png`, `.jpg`, `.jpeg` | `image/png`, `image/jpeg` | OCR parser | Extract page-level text, dimensions, OCR confidence, and orientation metadata. |

Image support is treated as a first-class OCR path, not as a generic binary fallback. The intake layer validates extension and MIME agreement, file signature, size, dimensions, and corruption before routing PNG/JPEG/JPG files to OCR.

## How We Will Build It

Open-source-first stack:

| Layer | Technology |
| --- | --- |
| API | Python FastAPI |
| Job processing | Celery/RQ or Kafka/Redpanda worker model |
| Parsing core | Docling, Apache Tika, Unstructured, PyMuPDF, python-docx, openpyxl, Pandoc, Pillow |
| OCR | Tesseract or PaddleOCR for scanned PDFs, PNG, JPEG, and JPG |
| Email parsing | mailparser or Python email package |
| Object storage | MinIO or S3-compatible storage |
| Metadata store | PostgreSQL for compact metadata, ACLs, job definitions, final status, and artifact pointers |
| Active state | Redis for hot job progress, locks, heartbeats, and retry metadata |
| Search output | OpenSearch or downstream RAG index |
| Observability | OpenTelemetry, Prometheus, Grafana, structured logs |
| Delivery | Docker Compose locally, Kubernetes/OpenShift for production-style deployment |

The platform is a modular monorepo with separately deployable services and parser workers. The codebase can share contracts and utilities, but the runtime is intentionally split because document parsers have very different dependency, CPU, memory, GPU, and failure-isolation needs.

Do not bundle every parser into one all-in-one container. Apache Tika brings a JVM, Pandoc brings system binaries, OCR engines can bring large native/ML runtimes, and PDF/layout tooling can become memory-heavy. The production design uses specialized workers:

| Worker | Handles | Runtime profile |
| --- | --- | --- |
| `worker-light` | TXT, Markdown, HTML, small clean DOCX/CSV | Small CPU container, high concurrency. |
| `worker-office` | DOCX, PPTX, XLSX, EML | Office/text libraries, medium CPU and memory limits. |
| `worker-pdf-native` | Born-digital PDFs | PyMuPDF/Docling-style PDF stack, medium CPU. |
| `worker-ocr` | Scanned PDFs, PNG, JPG, JPEG | Tesseract/PaddleOCR, high CPU/GPU optional, strict limits. |
| `worker-table` | Complex tables and table repair | Lower concurrency, quality-focused processing. |

Before routing, every file goes through a pre-flight inspector that checks MIME, magic bytes, file size, page count, text density, image density, encryption, corruption, and image dimensions. This prevents a fake extension or scanned PDF from being sent to the wrong parser. For example, a large scanned PDF should be routed to `queue_ocr`, not to native PyMuPDF extraction where it may return empty text or exhaust worker memory.

Work is routed to separate queues so fast text jobs are not blocked behind heavy OCR or table-repair jobs. The queue layer is treated as a resource-control boundary, not just an async transport:

```text
queue_light   -> worker-light
queue_office  -> worker-office
queue_pdf     -> worker-pdf-native
queue_ocr     -> worker-ocr
queue_table   -> worker-table
```

Each queue has its own concurrency, timeout, retry, dead-letter, and autoscaling policy. A 1 KB TXT file should never wait behind a 500-page scanned PDF.

PostgreSQL is not used as a blob store or high-frequency progress store. Large normalized outputs, intermediate chunks, page artifacts, full quality reports, and detailed lineage are written to MinIO/S3. Redis holds short-lived active pipeline state with TTLs. PostgreSQL stores durable records and pointers.

## Design Principles

- **Preserve structure**: text without layout is often not enough.
- **Route by document condition**: scanned PDFs, born-digital PDFs, spreadsheets, and emails need different paths.
- **Keep parser lineage**: every output should record parser, version, timestamp, source hash, and confidence.
- **Make failures visible**: silent empty extraction is worse than a hard failure.
- **Normalize outputs**: downstream systems should not care which parser handled the file.
- **Support RAG preparation**: extracted content should include stable source references and chunking hints.
- **Treat images safely**: image files must pass MIME, magic-byte, dimension, and corruption checks before OCR.
- **Isolate heavy dependencies**: parser runtimes should be split into specialized worker images instead of one bloated container.
- **Separate workload classes**: light text files and heavy OCR/table jobs should not compete in the same queue.

## Output Model

Each parsed document should produce:

```text
document.json      normalized structure
document.md        human-readable extracted content
document.txt       plain text fallback
tables/*.json      extracted table structures
metadata.json      file, parser, quality, and lineage metadata
```

Tables are normalized through sanitized HTML `<table>` as the canonical internal representation before JSON or Markdown export. HTML gives the platform a practical way to preserve `rowspan`, `colspan`, merged cells, nested headers, and multi-line table content across PDF, OCR, Excel, and document parsers. Markdown is generated only when it can represent the table without major structure loss.

For image inputs, the normalized output should include:

```text
image.width
image.height
image.format
image.orientation
ocr.engine
ocr.language
ocr.confidence
blocks[].bbox
blocks[].text
blocks[].confidence
```

## Project Structure

```text
open-document-intelligence-platform/
├── apps/
│   ├── api/
│   ├── worker-light/
│   ├── worker-office/
│   ├── worker-pdf-native/
│   ├── worker-ocr/
│   └── worker-table/
├── packages/
│   ├── core/
│   ├── detection/
│   ├── normalization/
│   ├── storage/
│   └── observability/
├── README.md
├── plan.md
├── diagrams/
│   └── document-intelligence-architecture.md
├── docs/
│   ├── architecture/
│   │   ├── preflight-inspector.md
│   │   ├── parser-routing-matrix.md
│   │   ├── worker-and-queue-strategy.md
│   │   ├── table-normalization.md
│   │   └── storage-and-lineage.md
│   └── adr/
│       ├── 0001-specialized-parser-workers.md
│       ├── 0002-isolated-workload-queues.md
│       ├── 0003-html-table-normalization.md
│       └── 0004-storage-and-lineage-boundaries.md
```

## Current Status

This project starts with planning and architecture. The next step is to implement file intake, parser routing, and the first PDF/DOCX/TXT parsing vertical slice.
