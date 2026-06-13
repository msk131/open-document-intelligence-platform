# Packages

Shared libraries live here.

Packages must stay lightweight and avoid pulling heavy parser runtime dependencies into the API or unrelated workers.

| Package | Purpose |
| --- | --- |
| `core` | Shared domain models, job states, errors, result contracts. |
| `detection` | MIME, magic-byte, preflight, and routing signals. |
| `normalization` | Common document model and artifact schemas. |
| `storage` | Object storage and artifact pointer abstractions. |
| `observability` | Logging, metrics, tracing, and correlation helpers. |
