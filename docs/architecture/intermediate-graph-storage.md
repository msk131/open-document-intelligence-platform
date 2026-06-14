# Intermediate Graph Storage

Some normalization outputs are too large and nested for ordinary PostgreSQL rows. Large structural matrices, page graphs, layout graphs, and chunk graphs should be stored as immutable artifacts or analytical partitions.

## Rule

PostgreSQL stores searchable summaries and pointers. Large graph-like structures are stored outside hot relational tables.

## Storage Options

| Data | Preferred storage |
| --- | --- |
| page layout graph | MinIO/S3 JSON artifact |
| block adjacency graph | MinIO/S3 JSON artifact |
| chunk graph | MinIO/S3 artifact or analytical partition |
| table structure matrix | canonical HTML + metadata artifact |
| searchable summary | PostgreSQL compact columns |
| operational analytics | partitioned analytical store when needed |

## Why

Large nested JSON structures cause:

- row bloat
- slow backups
- expensive indexes
- poor write throughput
- hard migrations

Object storage and analytical partitions are better suited for append-only structural artifacts.
