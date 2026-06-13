# ADR 0003: HTML Table Normalization

## Status

Accepted.

## Context

Different parsers expose tables in incompatible ways:

- Excel parsers expose sheets, rows, columns, formulas, and merged ranges.
- PDF/layout parsers expose bounding boxes, inferred cells, and reading order.
- OCR parsers expose text regions, coordinates, and confidence scores.
- DOCX/PPTX parsers expose document-native table structures and styles.

Forcing all of these directly into one rigid JSON schema causes either data loss or excessive parser-specific complexity.

## Decision

Use sanitized HTML `<table>` syntax as the canonical internal table representation.

The platform will generate:

- canonical HTML table artifact
- metadata JSON with lineage, quality, source location, and warnings
- derived JSON export for downstream APIs
- Markdown export only when the table can be represented without major structure loss

## Consequences

Benefits:

- Preserves `rowspan` and `colspan`.
- Handles grouped and multi-line headers better than Markdown.
- Gives Excel, PDF, OCR, DOCX, and PPTX adapters a shared target.
- Makes lossy Markdown export explicit.
- Keeps parser uncertainty in metadata instead of hiding it.

Costs:

- Requires HTML sanitization.
- Requires table validation.
- Requires derived JSON export logic.
- Consumers must understand that JSON is an export, while HTML is the canonical table structure.

This tradeoff is accepted because structure preservation is more important than forcing early JSON uniformity.
