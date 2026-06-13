# Open Document Intelligence Platform Plan

This plan defines how to build a production-grade document parsing platform with open-source tools.

## Phase 0: Why

The platform exists to turn heterogeneous documents into clean, structured, traceable data.

Business reasons:

- Reduce manual document review.
- Prepare documents for RAG and search.
- Extract tables and metadata for analytics.
- Improve auditability of document-derived data.
- Standardize document processing across file types.

Engineering reasons:

- Demonstrates robust ingestion and parsing architecture.
- Handles real-world file messiness.
- Creates a reusable parser layer for RAG and AI systems.
- Shows quality controls, retries, and traceable outputs.

## Phase 1: What

Initial file types:

- PDF: born-digital and scanned.
- DOCX.
- PPTX.
- XLSX and CSV.
- HTML and Markdown.
- TXT.
- EML emails.
- PNG, JPG, and JPEG images for OCR.

Initial outputs:

- Normalized JSON structure.
- Markdown extraction.
- Plain-text extraction.
- Table JSON.
- Metadata and quality report.

## Phase 2: How

Core modules:

| Module | Responsibility |
| --- | --- |
| intake | Upload, file validation, hashing, object storage. |
| preflight | MIME detection, extension mismatch checks, magic-byte checks, page sampling, text density, image density, encryption/corruption checks. |
| routing | Select parser strategy by file type and condition. |
| parsing | Extract text, layout, tables, and metadata. |
| ocr | OCR scanned PDFs, PNG, JPG, and JPEG images. |
| normalization | Convert parser-specific output into a common document model. |
| quality | Score extraction quality and flag failures. |
| export | Write JSON, Markdown, text, and table artifacts. |
| operations | Job state, retries, dashboards, and replay. |

Core stores:

| Store | Purpose |
| --- | --- |
| PostgreSQL | Documents, parse jobs, ACLs, final status, parser versions, compact quality scores, artifact pointers. |
| MinIO/S3 | Original files, derived files, extracted artifacts, intermediate chunks, full lineage, full quality reports. |
| Redis | Active job state, progress, locks, heartbeats, short-lived retry metadata. |
| Redis/Kafka | Background parsing queue or event stream. |

Runtime services and workers:

| Runtime | Responsibility | Queue |
| --- | --- | --- |
| api | Intake, job creation, status, artifact access, auth, quotas. | n/a |
| worker-light | TXT, Markdown, HTML, small CSV, simple clean files. | `queue_light` |
| worker-office | DOCX, PPTX, XLSX, EML and attachment extraction. | `queue_office` |
| worker-pdf-native | Born-digital PDF parsing and layout extraction. | `queue_pdf` |
| worker-ocr | Scanned PDFs, PNG, JPG, JPEG, OCR-heavy files. | `queue_ocr` |
| worker-table | Table repair, table normalization, multi-page table handling. | `queue_table` |

## Phase 3: Analyze

Key risks and controls:

| Risk | Control |
| --- | --- |
| Wrong parser selected | MIME detection, content sniffing, parser routing tests. |
| Scanned PDF routed to native parser | Preflight samples PDF pages for text density and image density before queue selection. |
| Bloated parser container | Split parser engines into specialized worker images with separate dependency stacks. |
| Empty extraction | Native parsers cannot mark near-empty output as success; quality gates trigger OCR fallback or failure. |
| Table corruption | Use canonical sanitized HTML table representation, table confidence score, structured validation, and export preview. |
| Rigid table JSON loses structure | Treat JSON/Markdown as exports from canonical HTML, not the source of truth. |
| Scanned document failure | OCR path and page-level confidence. |
| Light jobs stuck behind heavy jobs | Separate queues by workload class and set worker-specific concurrency limits. |
| Queue gridlock from uneven job sizes | Use isolated queues with per-queue concurrency, timeouts, retries, dead letters, and autoscaling. |
| Expensive OCR/VLM compute | Make OCR/VLM optional, route only when preflight proves it is needed, and enforce quotas/timeouts. |
| Image spoofing or malformed image | Validate MIME, extension, magic bytes, dimensions, decoder health, and max pixels. |
| Large file pressure | Size limits, streaming, worker isolation, timeout policy. |
| Sensitive data exposure | No secrets in logs, object storage access control, optional redaction hooks. |
| Non-repeatable output | Parser versioning, source file hash, deterministic artifact naming. |
| PostgreSQL overloaded by lineage/chunks | Keep PostgreSQL for compact metadata and pointers; store active state in Redis and large artifacts in MinIO/S3. |
| Redis state loss | Treat Redis as transient only; recover from PostgreSQL final records and object storage artifacts. |
| Compliance and private data handling | Support air-gapped deployment, tenant isolation, audit logs, retention policies, and optional redaction. |
| Open-source sustainability | Keep the local stack runnable without GPUs and make advanced OCR/VLM engines pluggable. |

## Phase 4: Act

### Slice 1: Project Foundation

- Create FastAPI project.
- Add document upload API.
- Add PostgreSQL and object storage through Docker Compose.
- Add parse job table and health endpoint.
- Create modular monorepo structure with `apps/` for deployables and `packages/` for shared contracts.
- Add artifact pointer model instead of storing extracted blobs in PostgreSQL.

Done when:

- A file can be uploaded, hashed, stored, and tracked.
- The API can run without installing every parser dependency.
- PostgreSQL stores metadata and pointers only for source artifacts.

### Slice 2: Type Detection And Routing

- Detect MIME type.
- Detect extension mismatch.
- Detect encrypted/corrupt files.
- Add pre-flight sampling for PDFs to estimate text density, page count, image density, and OCR need.
- Add parser routing matrix for native PDF, scanned PDF, mixed PDF, image OCR, Office, text, and reject/quarantine paths.
- Route PDF, DOCX, TXT, CSV, PNG, JPG, and JPEG to parser strategies.
- Validate image magic bytes and reject unsupported image formats.

Done when:

- Files are classified and routed with test coverage, including `.png`, `.jpg`, and `.jpeg`.
- Born-digital PDFs and scanned PDFs are routed differently before full parsing.
- A scanned PDF is not routed to `worker-pdf-native` based on MIME type alone.

### Slice 3: First Parsers

- Implement PDF text extraction.
- Implement DOCX extraction.
- Implement TXT and Markdown extraction.
- Emit normalized JSON, Markdown, and text.
- Run light, office, and PDF parsing in separate worker runtimes.

Done when:

- Three document types produce normalized output with metadata.
- Parser dependencies are isolated by worker image.

### Slice 4: OCR And Layout

- Add OCR path for scanned PDFs, PNG, JPG, and JPEG images.
- Add page-level extraction metadata.
- Add image-level metadata: dimensions, format, orientation, OCR language, and confidence.
- Add layout blocks and reading order.

Done when:

- Scanned documents and image files produce text with confidence scores.

### Slice 4A: Image Intake Hardening

- Enforce image size and max-pixel limits.
- Detect decompression-bomb risk.
- Normalize EXIF orientation before OCR.
- Store the original image and parser-safe derived image separately.
- Reject extension/MIME mismatches such as `.jpg` files with non-JPEG magic bytes.

Done when:

- PNG/JPG/JPEG uploads are accepted only when validated, routed to OCR, and represented in the normalized document model.

### Slice 5: Tables And Office Formats

- Add XLSX/CSV table extraction.
- Add table extraction from PDFs where possible.
- Add PPTX text and speaker-note extraction.
- Normalize tables through an internal HTML table representation before JSON/Markdown export.
- Add parser adapters for Excel, PDF layout, OCR table candidates, and DOCX/PPTX tables.
- Mark Markdown table export as lossy/skipped when row spans, column spans, nested tables, or OCR ambiguity are present.

Done when:

- Tables are exported as structured JSON artifacts.
- Merged cells, row spans, column spans, and multi-line headers are preserved where parsers expose them.
- Canonical table HTML, metadata JSON, and derived JSON artifacts are produced for complex tables.

### Slice 6: Quality And Operations

- Add quality report.
- Add failed-job retry.
- Add parser version tracking.
- Add dashboards for parse success rate, duration, file type, and low-confidence outputs.
- Add separate queue metrics for light, office, PDF, OCR, and table workers.
- Add timeout, retry, and dead-letter policies per queue.
- Add queue admission controls so saturated heavy queues do not affect light parsing.
- Add Redis-backed active job progress and S3-backed full lineage artifacts.
- Add reconciliation for stale Redis jobs and orphaned staged artifacts.

Done when:

- Parsing failures and low-quality outputs are visible and replayable.
- Heavy OCR/table jobs cannot starve light parsing jobs.
- `queue_light` remains responsive while `queue_ocr` or `queue_table` is saturated.
- PostgreSQL is not updated for every page/chunk/progress tick.
