# Normalization Package

Shared normalized document contracts.

Owns:

- document schema
- page schema
- block schema
- table artifact schema
- quality report schema
- lineage schema

## Table Normalization Contract

Tables use sanitized HTML `<table>` as the canonical internal structure.

The table contract should include:

- canonical HTML artifact path
- metadata JSON artifact path
- source document/page/sheet reference
- optional bounding box
- parser and adapter lineage
- confidence score
- span statistics
- warnings
- lossy export flags

JSON and Markdown are derived exports. Markdown should be skipped or marked lossy when the canonical table contains row spans, column spans, nested tables, grouped headers, or ambiguous OCR cells.
