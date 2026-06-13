# ADR 0001: Specialized Parser Workers

## Status

Accepted.

## Context

Document intelligence platforms need many parser engines, but those engines do not share the same runtime profile.

Examples:

- Apache Tika requires a JVM.
- OCR engines may require large native packages, CPU-heavy processing, or optional GPU support.
- PaddleOCR and layout-aware models can pull large ML dependencies.
- Pandoc and office tooling add system-level binaries.
- PDF parsers and table extractors can be memory intensive.

Putting every parser dependency into one API or Celery worker image creates a large, fragile runtime. It increases build time, startup time, CVE surface area, memory usage, and the blast radius of parser crashes.

## Decision

Use a modular monorepo for shared development, but deploy parser runtimes as specialized services/workers.

Initial deployables:

| Deployable | Purpose |
| --- | --- |
| `api` | Upload, job creation, status, artifact access, auth, quotas. |
| `worker-light` | TXT, Markdown, HTML, small CSV, simple clean files. |
| `worker-office` | DOCX, PPTX, XLSX, EML and attachment handling. |
| `worker-pdf-native` | Born-digital PDF parsing and layout extraction. |
| `worker-ocr` | Scanned PDFs, PNG, JPG, JPEG, OCR-heavy files. |
| `worker-table` | Complex table extraction, repair, and normalization. |

Shared code belongs in `packages/`, but heavy dependencies stay in the worker that needs them.

## Consequences

Benefits:

- Smaller service images.
- Better security patching per runtime.
- Independent scaling for OCR-heavy workloads.
- Better failure isolation.
- Cleaner local development because contributors can run only the service they need.

Costs:

- More Dockerfiles and deployment manifests.
- More queue routing logic.
- More integration testing.
- More observability dimensions.

This tradeoff is accepted because parser dependency isolation is a core production concern for this platform.
