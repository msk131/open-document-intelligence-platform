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
- PNG/JPEG images for OCR.

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
| detection | MIME detection, extension mismatch checks, encryption/corruption checks. |
| routing | Select parser strategy by file type and condition. |
| parsing | Extract text, layout, tables, and metadata. |
| ocr | OCR scanned PDFs and images. |
| normalization | Convert parser-specific output into a common document model. |
| quality | Score extraction quality and flag failures. |
| export | Write JSON, Markdown, text, and table artifacts. |
| operations | Job state, retries, dashboards, and replay. |

Core stores:

| Store | Purpose |
| --- | --- |
| PostgreSQL | Documents, parse jobs, parser versions, quality scores, artifact metadata. |
| MinIO/S3 | Original files and extracted artifacts. |
| Redis/Kafka | Background parsing job queue. |

## Phase 3: Analyze

Key risks and controls:

| Risk | Control |
| --- | --- |
| Wrong parser selected | MIME detection, content sniffing, parser routing tests. |
| Empty extraction | Quality gates, minimum text threshold, OCR fallback. |
| Table corruption | Table confidence score, structured validation, export preview. |
| Scanned document failure | OCR path and page-level confidence. |
| Large file pressure | Size limits, streaming, worker isolation, timeout policy. |
| Sensitive data exposure | No secrets in logs, object storage access control, optional redaction hooks. |
| Non-repeatable output | Parser versioning, source file hash, deterministic artifact naming. |

## Phase 4: Act

### Slice 1: Project Foundation

- Create FastAPI project.
- Add document upload API.
- Add PostgreSQL and object storage through Docker Compose.
- Add parse job table and health endpoint.

Done when:

- A file can be uploaded, hashed, stored, and tracked.

### Slice 2: Type Detection And Routing

- Detect MIME type.
- Detect extension mismatch.
- Detect encrypted/corrupt files.
- Route PDF, DOCX, TXT, and CSV to parser strategies.

Done when:

- Files are classified and routed with test coverage.

### Slice 3: First Parsers

- Implement PDF text extraction.
- Implement DOCX extraction.
- Implement TXT and Markdown extraction.
- Emit normalized JSON, Markdown, and text.

Done when:

- Three document types produce normalized output with metadata.

### Slice 4: OCR And Layout

- Add OCR path for scanned PDFs and images.
- Add page-level extraction metadata.
- Add layout blocks and reading order.

Done when:

- Scanned documents produce text with confidence scores.

### Slice 5: Tables And Office Formats

- Add XLSX/CSV table extraction.
- Add table extraction from PDFs where possible.
- Add PPTX text and speaker-note extraction.

Done when:

- Tables are exported as structured JSON artifacts.

### Slice 6: Quality And Operations

- Add quality report.
- Add failed-job retry.
- Add parser version tracking.
- Add dashboards for parse success rate, duration, file type, and low-confidence outputs.

Done when:

- Parsing failures and low-quality outputs are visible and replayable.

