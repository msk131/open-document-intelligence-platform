# Worker Office

Handles office and email formats.

Target inputs:

- DOCX
- PPTX
- XLSX
- EML
- supported attachments routed back through preflight

Runtime profile:

- Medium CPU and memory.
- Office-specific parsing dependencies.
- Optional Tika adapter if needed, isolated from the API.
