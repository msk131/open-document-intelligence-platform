# Compliance And Security

Enterprise deployments often require that documents never leave a controlled cloud, VPC, or on-premises boundary.

## Deployment Posture

The platform should support:

- on-premises deployment
- private VPC deployment
- air-gapped deployment
- no external model calls by default
- private object storage
- private container registry
- private package/model mirrors
- tenant-scoped encryption and retention

Open-source dependencies such as Docling, PaddleOCR, Tika, PyMuPDF, and Tesseract should run inside the customer boundary.

## Tenant Isolation

Tenant isolation should apply to:

- object storage prefixes/buckets
- PostgreSQL tenant columns or schemas
- queue admission quotas
- Redis key prefixes
- artifact access policies
- audit logs
- encryption keys when supported

Recommended source object path:

```text
s3://{tenant_id}/inputs/{yyyy}/{mm}/{dd}/{source_hash}.{ext}
```

## PII Redaction And Sensitive-Data Flagging

PII detection should run after deterministic text extraction and before open search/RAG indexing.

Sensitive data classes:

- payment card patterns
- national/social identifiers
- confidential legal names when policy configured
- addresses
- phone numbers
- email addresses
- account identifiers
- health or financial terms when policy configured

Redaction outputs must be idempotent:

- redaction policy version
- detector/model version
- input artifact hash
- output artifact hash
- decision summary

## Indexing Rule

Only policy-approved artifacts should be sent to OpenSearch or RAG indexes.

```text
raw extraction
  -> quality gate
  -> sensitive-data detection/redaction
  -> approved index artifact
  -> search/RAG
```

## Anti-Patterns

- Sending customer documents to external APIs by default.
- Indexing raw extracted text before PII checks.
- Sharing object prefixes across tenants.
- Logging extracted text or secrets.
- Treating redaction as a non-idempotent side effect.
