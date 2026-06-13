# Infra

Local and production-style deployment assets live here.

Planned assets:

- `docker-compose.yml` for API, PostgreSQL, MinIO, queue backend, and selected workers.
- `docker-compose.light.yml` for API plus light parsing only.
- `docker-compose.full.yml` for all workers including OCR.
- Kubernetes or Helm manifests for production-style deployment.

The infra design must preserve worker isolation. The API image should not include OCR, PDF, office, or table-repair dependencies unless explicitly running an all-in-one development profile.
