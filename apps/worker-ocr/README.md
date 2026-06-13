# Worker OCR

Handles OCR-heavy workloads.

Target inputs:

- Scanned PDFs
- PNG
- JPG
- JPEG
- mixed PDFs requiring OCR fallback

Runtime profile:

- High CPU.
- Optional GPU profile.
- Low concurrency.
- Strict timeout, page-count, and max-pixel limits.

This worker owns OCR dependencies such as Tesseract, PaddleOCR, Pillow, and image preprocessing libraries.
