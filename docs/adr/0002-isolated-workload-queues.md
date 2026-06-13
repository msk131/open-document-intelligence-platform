# ADR 0002: Isolated Workload Queues

## Status

Accepted.

## Context

Document parsing jobs have highly variable resource profiles.

Examples:

- A 1 KB text file can finish in milliseconds.
- A small DOCX may finish quickly but require Office-specific dependencies.
- A 500-page scanned PDF can run OCR for minutes.
- A table-heavy financial report can saturate CPU and memory.

A single generic queue can cause starvation. With simple round-robin scheduling, heavy jobs can occupy workers while small jobs wait behind them.

## Decision

Use separate workload queues:

| Queue | Purpose |
| --- | --- |
| `queue_light` | small text-like files |
| `queue_office` | Office and email files |
| `queue_pdf` | born-digital PDFs |
| `queue_ocr` | scanned PDFs and images |
| `queue_table` | table repair and complex table extraction |

Preflight assigns a workload class before enqueueing. Workers subscribe to the queue that matches their runtime profile.

## Consequences

Benefits:

- Light jobs do not wait behind OCR jobs.
- Worker concurrency can be tuned per workload.
- Timeouts and retries can be set per queue.
- Autoscaling can use queue-specific backlog.
- Costly OCR/VLM workloads can be quota-controlled.

Costs:

- More queue configuration.
- More deployment manifests.
- More metrics and alerts.
- More integration tests for routing correctness.

The tradeoff is accepted because predictable resource isolation is required for production-grade document processing.
