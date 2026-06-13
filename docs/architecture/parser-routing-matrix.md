# Parser Routing Matrix

Parser routing must use document condition, not MIME type alone. A `.pdf` can be a born-digital report, a scanned book, a mixed document, or a corrupt file with a fake extension.

## Routing Rules

| Input condition | Preflight signals | Queue | Worker | Primary parser path |
| --- | --- | --- | --- | --- |
| Small TXT/Markdown/HTML | text MIME, small size, safe encoding | `queue_light` | `worker-light` | text parser |
| Small CSV | CSV MIME, safe delimiter/encoding, size below limit | `queue_light` | `worker-light` | CSV parser |
| DOCX/PPTX/XLSX | Office MIME and ZIP/OpenXML signature | `queue_office` | `worker-office` | office parser |
| EML | `message/rfc822`, valid headers | `queue_office` | `worker-office` | email parser |
| Born-digital PDF | PDF signature, selectable text on sampled pages | `queue_pdf` | `worker-pdf-native` | PyMuPDF/Docling PDF path |
| Scanned PDF | PDF signature, low text density, high image density | `queue_ocr` | `worker-ocr` | OCR path |
| Mixed PDF | sampled pages disagree on native text availability | `queue_ocr` initially | `worker-ocr` | mixed/OCR-safe path |
| Table-heavy PDF | PDF table candidates or dense grid/layout signals | `queue_pdf`, then `queue_table` | `worker-pdf-native`, `worker-table` | PDF extraction plus table repair |
| PNG/JPG/JPEG | image MIME, valid magic bytes, safe dimensions | `queue_ocr` | `worker-ocr` | image OCR path |
| Spoofed extension | extension and magic bytes disagree | none | none | reject/quarantine |
| Corrupt/encrypted | parser-safe probe fails or password required | none | none | reject/quarantine |
| Oversized document | file/page/pixel/memory estimate exceeds limit | none or heavy queue | none or controlled worker | reject/split/manual |

## Anti-Patterns

- Do not route PDFs by `application/pdf` alone.
- Do not send a scanned PDF to native PDF extraction without sampling.
- Do not let OCR jobs share a queue with small text jobs.
- Do not retry corrupt files endlessly.
- Do not treat empty extraction as success.

## Routing Contract

Every routing decision should persist:

- detected MIME
- extension
- magic-byte result
- classification
- sampled page indexes
- text density
- image density
- page count
- file size
- queue
- worker
- reason
- risk flags

This makes routing explainable, testable, and debuggable.
