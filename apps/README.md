# Apps

Deployable services and parser workers live here.

The platform uses a modular monorepo, but each app can become a separate container image with its own dependency stack, resource limits, queue subscription, and scaling policy.

| App | Purpose |
| --- | --- |
| `api` | Intake API, job creation, status, auth, quotas, artifact access. |
| `worker-light` | Fast parsing for TXT, Markdown, HTML, small CSV, and simple clean files. |
| `worker-office` | DOCX, PPTX, XLSX, EML, and attachment extraction. |
| `worker-pdf-native` | Born-digital PDF parsing and layout extraction. |
| `worker-ocr` | Scanned PDFs, PNG, JPG, JPEG, and OCR-heavy files. |
| `worker-table` | Complex table extraction, repair, and normalization. |
