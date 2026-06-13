# Detection Package

Shared preflight and routing signal code.

Owns:

- MIME detection interfaces
- magic-byte checks
- extension mismatch rules
- PDF sampling contracts
- image safety contracts
- routing decision models

## Preflight Contract

This package should provide the shared contract used by the API, queue router, and workers:

```text
PreflightResult
  document_id
  source_hash
  detected_mime
  extension
  magic_byte_status
  file_size_bytes
  page_count
  sampled_pages
  text_density
  image_density
  classification
  recommended_queue
  recommended_worker
  routing_reason
  risk_flags
```

The implementation must make it hard to route by MIME type alone. PDF routing should always consider sampled page signals before selecting `worker-pdf-native` or `worker-ocr`.
