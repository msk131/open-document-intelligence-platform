# Worker And Queue Strategy

Document parsing workloads are uneven. A 2 KB text file may finish in milliseconds, while a 500-page scanned PDF can consume CPU for minutes. The platform uses separate queues and workers so heavy jobs do not block light jobs.

The queue layer is a resource-control boundary. It should prevent slow OCR, large PDFs, and complex tables from starving small text jobs.

## Queues

| Queue | Worker | Workload | Concurrency | Timeout posture |
| --- | --- | --- | --- | --- |
| `queue_light` | `worker-light` | TXT, Markdown, HTML, small CSV | High | Short |
| `queue_office` | `worker-office` | DOCX, PPTX, XLSX, EML | Medium | Medium |
| `queue_pdf` | `worker-pdf-native` | Born-digital PDFs | Medium | Medium/long |
| `queue_ocr` | `worker-ocr` | Scanned PDFs, PNG, JPG, JPEG | Low | Long |
| `queue_table` | `worker-table` | Complex tables and repair | Low | Long |

## Workload Classes

Preflight assigns each job a workload class before it enters the queue system.

| Class | Examples | Queue |
| --- | --- | --- |
| `light` | 1 KB TXT, Markdown, small HTML, small CSV | `queue_light` |
| `office` | DOCX, PPTX, XLSX, EML | `queue_office` |
| `pdf_native` | born-digital PDFs with selectable text | `queue_pdf` |
| `ocr_heavy` | scanned PDFs, image uploads, low text-density PDFs | `queue_ocr` |
| `table_heavy` | multi-page tables, nested tables, repair candidates | `queue_table` |
| `oversized` | over configured file/page/pixel limits | reject, split, or controlled heavy path |

This classification prevents a large OCR job from being inserted into a generic queue where it can block simple files.

## Admission Control

The API should not blindly enqueue every accepted upload.

- Reject documents over hard safety limits.
- Mark documents over normal limits for split/manual processing.
- Apply tenant or user quotas before queueing.
- Assign queue from preflight classification.
- Persist routing reason and queue assignment.
- Enforce per-queue max in-flight jobs.
- Return a clear status when a queue is saturated.

Admission control is especially important for public demos where OCR can become the dominant infrastructure cost.

## Celery/RQ vs Kafka/Redpanda

The first implementation can use Celery/RQ because it is simple and fits a job-processing workflow. Kafka/Redpanda becomes useful when the platform needs durable event streams, replay, higher fan-out, or downstream consumers.

| Choice | Good for | Risks |
| --- | --- | --- |
| Celery/RQ | Simple job queues, fast local development, direct worker model. | Poor generic queue design can create round-robin gridlock or worker starvation. |
| Kafka/Redpanda | Durable event stream, replay, high-scale consumers, audit/event pipelines. | More operational complexity and requires careful consumer-group design. |

Regardless of backend, the platform must preserve separate workload queues/topics.

Suggested mapping:

```text
Celery/RQ queues:
  queue_light
  queue_office
  queue_pdf
  queue_ocr
  queue_table

Kafka/Redpanda topics:
  document.parse.light
  document.parse.office
  document.parse.pdf
  document.parse.ocr
  document.parse.table
```

## Anti-Starvation Controls

Avoid one shared worker pool consuming every workload class.

- Workers should subscribe only to their own queue unless explicitly configured for fallback.
- `worker-light` should never consume OCR jobs.
- `worker-ocr` should run at low concurrency with strict memory and CPU limits.
- Heavy queues should have independent autoscaling policies.
- Queue depth and age must be measured per queue.
- The scheduler should alert when light job age grows while heavy queues are saturated.
- Retries should return to the same workload queue, not a global retry queue.

## Runtime Isolation

Each worker image owns only the dependencies required for its workload.

| Worker | Example dependencies |
| --- | --- |
| `worker-light` | Markdown/HTML/text parsers. |
| `worker-office` | `python-docx`, `openpyxl`, email parser, optional Tika adapter. |
| `worker-pdf-native` | PyMuPDF, Docling or equivalent PDF/layout tooling. |
| `worker-ocr` | Tesseract or PaddleOCR, Pillow, image preprocessing libraries. |
| `worker-table` | Table repair libraries, HTML table normalization, validators. |

## Retry Policy

Retries must be per queue, not global.

| Queue | Retry posture | Dead-letter condition |
| --- | --- | --- |
| `queue_light` | Short retry, quick failure. | repeated parser error or unsupported encoding |
| `queue_office` | Retry parser crashes once or twice. | corrupt Office container or password-protected file |
| `queue_pdf` | Retry transient failures, avoid endless corrupt-file loops. | corrupt PDF, invalid xref, repeated empty output |
| `queue_ocr` | Low retry count, long timeout, dead-letter on repeated failure. | OCR timeout, unsupported image, memory limit exceeded |
| `queue_table` | Retry only deterministic repair stages. | invalid table model or repeated repair failure |

## Scaling

- API scales by request volume.
- Light workers scale by queue depth.
- OCR workers scale by CPU/GPU availability and budget.
- Table workers scale by backlog and quality SLA.

The platform should expose metrics per queue: depth, age of oldest job, success rate, failure rate, retry count, timeout count, and average processing duration.

## Metrics And Alerts

Minimum queue metrics:

- queue depth
- age of oldest job
- active workers
- in-flight jobs
- success count
- failure count
- retry count
- dead-letter count
- timeout count
- p50/p95/p99 processing duration
- worker CPU and memory by queue

Useful alerts:

- `queue_light` oldest job age exceeds target.
- `queue_ocr` timeout rate spikes.
- dead-letter count increases for a parser version.
- worker memory limit exits increase.
- retry count grows without corresponding success.

## Anti-Patterns

- One generic `parse_documents` queue for every file type.
- One Celery worker image with all dependencies and all queues.
- Round-robin scheduling across wildly different job sizes.
- A global retry queue that mixes light and heavy jobs.
- Autoscaling only on total queue depth instead of per-queue backlog.
