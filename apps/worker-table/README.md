# Worker Table

Handles complex table extraction and repair.

Target inputs:

- PDF table candidates.
- OCR table candidates.
- XLSX/CSV table normalization.
- DOCX/PPTX table extraction output.

Runtime profile:

- Lower concurrency.
- Quality-focused processing.
- HTML table intermediate representation before JSON/Markdown export.

Responsibilities:

- Convert parser-specific table candidates into sanitized canonical HTML tables.
- Preserve `rowspan`, `colspan`, captions, grouped headers, and multi-line cells.
- Attach source metadata such as page, sheet, bounding box, parser, and confidence.
- Validate spans, empty tables, OCR confidence, and source references.
- Generate JSON exports from canonical HTML.
- Generate Markdown only when the structure can be represented without major loss.
