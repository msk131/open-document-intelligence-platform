# SLO And Capacity Planning

The platform should define SLOs by workload class. A single global latency target is misleading because text parsing, OCR, table repair, and LLM enrichment have very different runtime profiles.

## Starter SLO Placeholders

These are initial design targets, not final benchmarks.

| Stage | Starter target |
| --- | --- |
| API upload accepted | p95 under 1 second after file transfer completes. |
| Preflight classification | p95 under 5 seconds for normal files. |
| Light parsing | p95 under 10 seconds. |
| Office parsing | p95 under 60 seconds. |
| Native PDF parsing | p95 under 120 seconds for normal PDFs. |
| OCR parsing | p95 measured by pages per minute, not only job latency. |
| Table repair | p95 under 180 seconds for normal table-heavy documents. |
| LLM enrichment | best-effort async; not required for base parse success. |

## Throughput Metrics

Measure throughput by workload:

- files per minute
- pages per minute
- OCR pages per minute
- tables per minute
- chunks per minute
- LLM requests per minute
- LLM tokens per minute

## Capacity Inputs

Each deployment should declare:

- max file size
- max page count
- max image pixels
- max concurrent OCR jobs
- max concurrent LLM jobs
- per-tenant queue limits
- per-tenant LLM token budget
- target queue age per workload class

## Benchmark Plan

Benchmark with separate fixtures:

- 1 KB TXT
- small DOCX
- small born-digital PDF
- 100-page born-digital PDF
- 100-page scanned PDF
- table-heavy financial PDF
- PNG/JPG photo with skew/blur
- multi-page table spanning pages
- LLM enrichment over source-anchored chunks

## SLO Rule

Base parsing SLOs must not depend on optional LLM enrichment. LLM enrichment has its own queue, budget, and completion target.
