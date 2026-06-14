# Infra

Local and production-style deployment assets live here.

Planned assets:

- `docker-compose.yml` for API, PostgreSQL, MinIO, queue backend, and selected workers.
- `docker-compose.light.yml` for API plus light parsing only.
- `docker-compose.full.yml` for all workers including OCR.
- Kubernetes or Helm manifests for production-style deployment.

The infra design must preserve worker isolation. The API image should not include OCR, PDF, office, or table-repair dependencies unless explicitly running an all-in-one development profile.

## Enterprise Deployment Profiles

Planned profiles:

- local CPU-only development
- full local stack with OCR
- private VPC deployment
- on-premises deployment
- air-gapped deployment
- GPU/VLM worker pool with scale-to-zero

Enterprise profiles should support private object storage, private model/package mirrors, private container registry, tenant-scoped secrets, and no external model calls by default.
