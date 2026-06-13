# Table Normalization

Table extraction differs heavily by source format.

Examples:

- Excel exposes explicit sheets, rows, columns, formulas, and merged cells.
- PDF parsers expose layout boxes and inferred cell boundaries.
- OCR engines expose text regions and confidence scores.
- DOCX/PPTX can expose nested tables, merged cells, and styled headers.

A single rigid JSON shape is difficult to preserve across all parser outputs.

## Decision

Use an internal HTML table representation before exporting JSON or Markdown. The internal representation is canonical for table structure; JSON and Markdown are export formats.

HTML tables naturally represent:

- `rowspan`
- `colspan`
- header cells
- nested or grouped headers
- multi-line cell content
- captions
- sections such as `thead`, `tbody`, and `tfoot`
- source-neutral structure across Excel, PDF, OCR, DOCX, and PPTX

## Why Not Rigid JSON First

Different parsers produce incompatible table signals:

| Source | Parser signal | Normalization challenge |
| --- | --- | --- |
| Excel/openpyxl | sheet, row, column, merged ranges, formulas, cell types | very structured, but not layout/bounding-box first |
| PDF/Docling | bounding boxes, inferred cells, reading order, page coordinates | structure is inferred and confidence-based |
| OCR/PaddleOCR | text boxes, coordinates, OCR confidence | table grid may be missing or ambiguous |
| DOCX/PPTX | XML table structures, merged cells, styles | may contain nested tables or styled headers |

Forcing all of these directly into one rigid `tables/*.json` shape too early creates lossy mappings and edge-case-heavy code. HTML is a practical intermediate because it can preserve the most common structural features while still being easy to validate and convert.

## Canonical Internal Table

The canonical table artifact should contain both:

- `table.html`: sanitized internal HTML table.
- `table.metadata.json`: parser lineage, source location, confidence, warnings, and export metadata.

Example:

```html
<table data-table-id="tbl_001" data-source-page="3">
  <caption>Quarterly exposure summary</caption>
  <thead>
    <tr>
      <th rowspan="2">Region</th>
      <th colspan="2">Exposure</th>
    </tr>
    <tr>
      <th>Current</th>
      <th>Previous</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>APAC</td>
      <td>1200000</td>
      <td>1150000</td>
    </tr>
  </tbody>
</table>
```

The HTML must be sanitized and restricted to a small allow-list:

```text
table, caption, thead, tbody, tfoot, tr, th, td, br
```

Allowed attributes should be limited to structural/source metadata:

```text
rowspan, colspan, data-*, scope
```

No scripts, inline event handlers, remote links, or style execution should be allowed.

## Parser Adapters

Each parser writes to the canonical HTML table model through an adapter.

| Adapter | Input | Output responsibility |
| --- | --- | --- |
| Excel adapter | rows, columns, merged cells, formulas, cell types | preserve row/column order, merged cells, sheet name, formulas as metadata |
| PDF layout adapter | bounding boxes, inferred cells, reading order | produce HTML plus page/bounding-box metadata |
| OCR table adapter | text regions, coordinates, confidence | produce best-effort HTML with low-confidence warnings |
| DOCX/PPTX adapter | document XML table nodes | preserve merged cells, headers, nested table warnings |

Adapters should not try to hide uncertainty. If a table is inferred, the artifact should say so.

## Processing Flow

```text
source parser output
  -> table candidate
  -> internal HTML table
  -> validation
  -> table metadata
  -> JSON artifact
  -> Markdown artifact when safe
```

## Export Rules

| Export | Rule |
| --- | --- |
| HTML | Canonical internal representation. Highest structure preservation. |
| JSON | Derived from canonical HTML and metadata. Good for APIs and downstream systems. |
| Markdown | Convenience output only when the table has no complex spans or nested structure. |
| Text | Fallback for search, not a structural table representation. |

Markdown should be skipped or marked lossy when the table contains:

- `rowspan`
- `colspan`
- nested tables
- multi-line headers
- ambiguous OCR cells
- table sections that cannot be represented cleanly

## JSON Shape

The JSON export should reference the canonical HTML rather than pretending to be the source of truth.

```json
{
  "table_id": "tbl_001",
  "canonical_format": "html-table",
  "html_artifact": "tables/tbl_001.html",
  "source": {
    "document_id": "uuid",
    "page": 3,
    "sheet": null,
    "bbox": [72, 144, 520, 420]
  },
  "structure": {
    "row_count": 12,
    "column_count": 5,
    "has_rowspan": true,
    "has_colspan": true,
    "has_nested_table": false
  },
  "quality": {
    "confidence": 0.82,
    "inferred_structure": true,
    "manual_review_recommended": false,
    "warnings": []
  }
}
```

## Validation

Table validation should check:

- row and cell consistency after expanding spans
- invalid or overlapping spans
- empty table detection
- header/body separation when possible
- OCR confidence thresholds
- source references and bounding boxes
- sanitized HTML allow-list compliance

## Quality Signals

Each table artifact should include:

- parser name and version
- source page or sheet reference
- bounding box when available
- confidence score
- row and column count
- merged-cell count
- warnings for inferred structure
- whether manual review is recommended
- whether Markdown export is lossy or skipped

Markdown export is a convenience output. The HTML intermediate and JSON artifact are the stronger structural outputs.
