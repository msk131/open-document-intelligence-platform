# Chunking And Context Reconstruction

Chunking must preserve source context. Layout-aware parsers can split paragraphs, tables, and sections across page boundaries, so chunks need enough anchors to be recombined.

## Design Rule

Chunks are derived artifacts, not the source of truth. The normalized document model remains the source for reconstruction.

Each chunk should carry:

- document ID
- source artifact hash
- page range
- section path
- block IDs
- table IDs
- bounding boxes when available
- reading-order indexes
- previous/next chunk pointers
- parser and chunker version

## Multi-Page Table Reconstruction

Multi-page tables should be reconstructed before LLM/RAG export when possible.

Signals:

- same table caption or nearby heading
- repeated header rows
- compatible column count
- compatible column labels
- continuation markers
- adjacent page positions
- stable table lineage IDs from parser/table worker

Flow:

```text
page table candidates
  -> table continuation detection
  -> canonical HTML table merge
  -> validation
  -> table artifact
  -> chunk references table artifact
```

Chunks should reference the merged table artifact instead of embedding partial table fragments as unrelated text.

## Paragraph And Section Continuation

Paragraphs split across pages should preserve:

- previous block ID
- next block ID
- paragraph group ID
- section heading path
- page start/end

When a paragraph is split at a page boundary, chunk generation should either merge it or mark the split with continuation metadata.

## LLM Context Packaging

LLM requests should be assembled from source-anchored chunks:

- include the section path
- include adjacent chunk context when needed
- include merged table artifacts instead of partial table text
- include citation/source references
- enforce token budgets before request creation

## Anti-Patterns

- Splitting purely by character count.
- Losing page and block IDs after chunking.
- Sending partial multi-page tables to LLMs as unrelated chunks.
- Treating chunks as authoritative when normalized source artifacts exist.
