# Pre-Flight Inspector

The pre-flight inspector runs before parser routing. Its purpose is to avoid choosing a parser based only on file extension or declared MIME type.

## Responsibilities

- Validate declared content type against detected MIME type.
- Validate file extension against magic bytes.
- Detect encrypted, password-protected, or corrupt files.
- Calculate file hash for deduplication and lineage.
- Enforce size, page-count, and max-pixel limits.
- Sample PDFs to estimate text density, image density, page complexity, and OCR need.
- Detect whether OCR is likely required.
- Select the queue and worker class before any full parser loads the document.
- Capture basic metadata for routing and audit.

## Routing Signals

| Signal | Used for |
| --- | --- |
| MIME type | Initial candidate parser selection. |
| Magic bytes | Spoofing and extension mismatch detection. |
| File size | Timeout and queue selection. |
| Page count | Worker class and resource limit selection. |
| Text density | PDF native parsing vs OCR routing. |
| Image density | OCR and layout-model routing. |
| Estimated memory cost | Reject, split, or route to low-concurrency heavy worker. |
| Estimated OCR cost | OCR queue selection and timeout budget. |
| Dimensions | Image safety and OCR cost estimation. |
| Encryption/corruption | Reject, quarantine, or manual review. |

## PDF Sampling

For PDFs, inspect a small page sample before full parsing. The inspector must not run the full extraction pipeline just to decide where the document belongs.

- First page.
- Last page for long documents.
- One middle page when page count is high.

For very large PDFs, the inspector should use bounded reads and stop early when limits are exceeded.

The inspector should estimate whether the document is:

- Born-digital PDF with selectable text.
- Scanned PDF requiring OCR.
- Mixed PDF with both text and scanned pages.
- Corrupt or encrypted PDF.

## PDF Classification

| Classification | Signals | Route |
| --- | --- | --- |
| `pdf_native` | Selectable text exists on sampled pages, text density is healthy, image density is not dominant. | `queue_pdf` -> `worker-pdf-native` |
| `pdf_scanned` | Little or no selectable text, page is mostly image content. | `queue_ocr` -> `worker-ocr` |
| `pdf_mixed` | Some sampled pages have text, others appear scanned. | `queue_ocr` or split-page plan when implemented. |
| `pdf_complex_table` | Table-heavy layout, dense grid lines, or many extracted table candidates. | `queue_pdf`, then optional `queue_table` |
| `pdf_too_large` | Page count, file size, or estimated memory cost exceeds configured limits. | reject, split, or route to controlled heavy queue |
| `pdf_invalid` | Corrupt, encrypted without password, or spoofed extension. | reject or quarantine |

## Suggested Initial Thresholds

These are starting values for v1 and should become configuration, not constants buried in code.

| Setting | Suggested default |
| --- | --- |
| Max upload size | 100 MB locally, lower for public demo deployments. |
| Max PDF pages | 500 pages before split/reject policy. |
| PDF sample pages | first, middle, last. |
| Native text threshold | route native when sampled pages have meaningful selectable text. |
| OCR threshold | route OCR when sampled pages have near-zero selectable text and dominant image area. |
| Worker timeout | separate timeout per queue, with OCR/table queues longest. |
| Parser memory limit | configured per worker image, not globally. |

The exact scoring can evolve, but the routing decision must be based on sampled document condition rather than MIME type alone.

## Failure Behavior

Preflight should fail closed:

- Unsupported MIME or magic-byte mismatch: reject.
- Corrupt file: reject or quarantine.
- Encrypted file without password: reject with a clear reason.
- File above hard safety limits: reject before queueing.
- File above normal limits but under hard limits: route to controlled heavy queue or mark for manual/split processing.
- Ambiguous PDF: prefer OCR or mixed strategy over native-only parsing to avoid silent empty output.

## OOM Prevention

The inspector is the first guardrail against worker crashes.

- It must run with bounded reads.
- It must avoid loading the entire document into memory.
- It must estimate page count and size before queueing.
- It must route heavy documents to low-concurrency workers.
- It must record the reason for heavy routing in metadata.
- It must reject files that exceed hard platform limits.

The goal is to avoid sending a large scanned PDF to `worker-pdf-native`, where a native parser may return empty text or consume too much memory.

## Image Checks

For PNG, JPG, and JPEG:

- Validate `image/png` or `image/jpeg`.
- Verify magic bytes.
- Decode with a safe image library.
- Enforce max width, height, and total pixels.
- Detect decompression-bomb risk.
- Normalize EXIF orientation before OCR.

## Output

The inspector emits a routing decision input, not final parsed content:

```json
{
  "document_id": "uuid",
  "source_hash": "sha256",
  "detected_mime": "application/pdf",
  "extension": ".pdf",
  "page_count": 12,
  "text_density": 0.87,
  "image_density": 0.12,
  "classification": "pdf_native",
  "ocr_required": false,
  "recommended_queue": "queue_pdf",
  "recommended_worker": "worker-pdf-native",
  "routing_reason": "sampled_pages_contain_selectable_text",
  "risk_flags": []
}
```
