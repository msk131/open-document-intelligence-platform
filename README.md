# Open Document Intelligence Platform

This repository documents a minimal design for an enterprise document intelligence platform. The focus is file intake, preflight inspection, parser routing, OCR fallback, normalized output, quality checks, lineage, and downstream RAG/search readiness.

## Design Focus

| Area | Design Point |
| --- | --- |
| File intake | Accept documents through API, batch import, or object storage triggers. |
| Preflight inspection | Check MIME type, magic bytes, size, page count, text density, encryption, corruption, and image dimensions. |
| Parser routing | Route by document condition, not extension alone. |
| Worker isolation | Separate light, office, PDF, OCR, table, and optional LLM workloads. |
| Normalization | Emit a common document model with text blocks, tables, metadata, confidence, and source anchors. |
| Quality gates | Fail or route low-confidence extraction to review before indexing. |
| Storage and lineage | Keep source files and large artifacts in object storage, with metadata and pointers in PostgreSQL. |

## Diagrams

| Diagram | Purpose |
| --- | --- |
| [diagrams/01-high-level-design.md](diagrams/01-high-level-design.md) | Platform-level document intelligence architecture. |
| [diagrams/02-low-level-design.md](diagrams/02-low-level-design.md) | Parser routing and extraction flow. |

## Current Status

Design-only repository. Implementation is intentionally out of scope until intake, routing, normalization, storage, and quality boundaries are clear.
