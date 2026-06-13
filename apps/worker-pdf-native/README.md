# Worker PDF Native

Handles born-digital PDFs where selectable text and layout information are available.

Target inputs:

- PDFs with sufficient text density.
- PDFs where preflight does not require full OCR.

Runtime profile:

- PDF/layout dependencies.
- Medium CPU and memory.
- Page-level timeouts and memory limits.

Scanned or mixed PDFs should be routed to `worker-ocr` when preflight indicates OCR is required.
