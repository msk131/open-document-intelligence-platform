# Enterprise Quality Guardrails

Silence is the enemy of enterprise automation. The platform must fail, flag, or route to review when extraction quality is not good enough.

## Quality Assertion Profile

Every normalized document should emit a quality profile:

```json
{
  "status": "VERIFIED",
  "score": 0.9214,
  "reason": null,
  "metrics": {
    "total_elements": 418,
    "character_density": 27492,
    "avg_confidence": 0.9214,
    "min_confidence": 0.77,
    "empty_pages": 0,
    "table_count": 12,
    "low_confidence_blocks": 3
  }
}
```

## Statuses

| Status | Meaning | Downstream behavior |
| --- | --- | --- |
| `VERIFIED` | confidence and structure checks passed | eligible for search/RAG/index export |
| `REVIEW_REQUIRED` | usable but below confidence or integrity threshold | route to review queue, index only if policy allows |
| `FAIL` | empty/corrupt/incomplete extraction | do not index; fail job or quarantine |

## Required Checks

- empty text detection
- element count
- page coverage
- average and minimum confidence
- table validation status
- OCR confidence distribution
- missing page detection
- parser fallback used
- PII redaction status
- artifact hash registration

## Empty Extraction Rule

If normalized output has no text and no structural elements:

```text
status = FAIL
reason = EMPTY_EXTRACTION_ALERT
score = 0.0
```

This output must not be silently indexed.

## Review Routing

Low-confidence or partially inferred outputs should be routed to a human-in-the-loop review queue with source anchors:

- document ID
- page number
- block/table ID
- bounding box
- confidence score
- parser/version
- failure reason
