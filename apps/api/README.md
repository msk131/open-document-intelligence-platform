# API Service

The API service handles intake and orchestration only. It should not install heavy parser dependencies.

Responsibilities:

- Upload and import documents.
- Validate request metadata.
- Create parse jobs.
- Store original files in object storage.
- Run basic intake validation.
- Expose job status and artifact access.
- Enforce auth, quotas, and tenant boundaries.

Heavy parsing must be delegated to specialized workers through queues.
