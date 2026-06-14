# Compute Tier Architecture

Enterprise document workloads must be profiled before execution. A flat pipeline can stall when a GPU-heavy scanned document lands beside thousands of clean text documents.

## Tiers

| Tier | Workloads | Workers | Scaling model |
| --- | --- | --- | --- |
| Tier-1 CPU | TXT, Markdown, HTML, CSV, DOCX, XLSX, EML, born-digital PDFs | `worker-light`, `worker-office`, `worker-pdf-native` | high replica count, low-cost CPU nodes |
| Tier-2 GPU/VLM | scanned PDFs, image OCR, layout detection, VLM-based extraction | `worker-ocr` with GPU/layout profile | dynamic GPU/spot pool, scale-to-zero when idle |
| Tier-3 Enrichment | table repair, LLM enrichment, redaction review | `worker-table`, `worker-llm` | budgeted concurrency and rate limits |

## Flow

```text
Enterprise API Gateway
  -> ingress file streaming
  -> fast profiling stage
  -> Tier-1 CPU queues for direct text/structure
  -> Tier-2 GPU/VLM queues for visual documents
  -> normalizer
  -> quality gates
  -> export/index
```

## Rules

- Preflight chooses compute tier before full parsing.
- Tier-1 and Tier-2 workers never share one consumer pool.
- GPU workers can scale to zero when idle.
- GPU queues must expose backlog, age, cost, and saturation metrics.
- Tier-2 jobs must have page/pixel/file limits before queue admission.
- Tier-1 throughput should remain healthy while Tier-2 is saturated.

## Anti-Patterns

- One flat queue for text, Office, PDF, OCR, VLM, and LLM jobs.
- Running OCR/VLM dependencies in Tier-1 workers.
- Keeping idle GPU replicas always warm in cost-sensitive deployments.
- Letting scanner dumps starve automated clean-email imports.
