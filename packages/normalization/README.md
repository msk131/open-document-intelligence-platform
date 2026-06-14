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

## Chunking Contract

Chunks must preserve enough source anchors to reconstruct context.

Each chunk should include:

- document ID
- source artifact hash
- page range
- section path
- block IDs
- table IDs
- bounding boxes when available
- reading-order indexes
- previous/next chunk pointers
- parser and chunker version

Chunks are derived artifacts. The normalized document and canonical table artifacts remain the source of truth.

## Quality Profile Contract

Normalization should emit a quality assertion profile:

```text
quality.status
quality.score
quality.reason
quality.metrics.total_elements
quality.metrics.character_density
quality.metrics.avg_confidence
quality.metrics.min_confidence
quality.metrics.empty_pages
quality.metrics.low_confidence_blocks
```

Empty extraction must produce `FAIL` with reason `EMPTY_EXTRACTION_ALERT`. Low-confidence output should produce `REVIEW_REQUIRED` unless policy explicitly allows indexing.
