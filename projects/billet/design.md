# Billet — Project Design Doc

*Status: Draft — Step 0 Ideation*
*Created: March 31, 2026*
*Authors: Dan Hannah & Clay*

---

## Overview

### What Is This?

Billet is a document ingestion pipeline that converts raw files (Word docs, PDFs, Excel, etc.) into clean markdown. It sits upstream of Anvil in the @claymore-dev ecosystem — Billet handles format conversion, Anvil handles search.

**The name:** A billet is a semi-finished piece of metal — raw material that's been shaped into a workable form but isn't a final product yet. That's exactly what this does: takes raw documents and shapes them into markdown that downstream tools can consume.

### The Pipeline

```
Raw Files (Word, PDF, Excel, HTML, etc.)
        ↓
  @claymore-dev/billet
  (format detection → extraction → cleaning → markdown output)
        ↓
  Clean Markdown Files
        ↓
  @claymore-dev/anvil (optional downstream)
  (chunks, embeds, semantic search)
```

### Why This Matters

Anvil is great at searching markdown. But most real-world documents aren't markdown:
- FIA technical regulations → dense PDFs with multi-column layouts, tables, cross-references
- Dan's mom's books → Word docs with complex formatting
- QuoteAI source material → could be PDFs, spreadsheets, web pages
- Enterprise docs → Word, PowerPoint, Confluence exports, etc.

Billet bridges the gap. It's the "make it markdown" step that enables everything downstream.

---

## Core Concepts

### Format Adapters

Each file type gets a dedicated adapter:

| Format | Adapter | Complexity | Notes |
|--------|---------|-----------|-------|
| Word (.docx) | mammoth.js | Low | Good markdown output natively |
| PDF | pdf-parse + custom | High | Multi-column, tables, images are hard |
| HTML | turndown | Low | HTML → markdown is well-solved |
| Excel (.xlsx) | xlsx + custom | Medium | Sheets → markdown tables |
| Plain text | passthrough | Trivial | Already close to markdown |
| PowerPoint | pptx + custom | Medium | Slide → section conversion |

### Cleaning Pipeline

Raw extraction isn't enough. Billet should:
1. **Detect format** — MIME type + extension
2. **Extract content** — format-specific adapter
3. **Clean** — normalize headings, fix encoding, remove artifacts
4. **Structure** — detect document hierarchy (chapters, sections, subsections)
5. **Output** — clean markdown with preserved structure

### Configuration

```yaml
# billet.config.yaml
input:
  path: ./raw-docs/
  formats: [docx, pdf, html]
  
output:
  path: ./markdown/
  
options:
  preserve_tables: true
  extract_images: false  # v2
  heading_detection: auto
  encoding: utf-8
```

---

## Use Cases

### FIA Technical Regulations (GMPPU)
- Dense regulatory PDFs with article numbering (1.1, 1.1.1, etc.)
- Multi-column layouts, inline tables, cross-references
- Need: reliable heading detection, table preservation, cross-reference linking
- Pipeline: `FIA PDFs → Billet → markdown → Anvil → Foundry/MCP search`

### Dan's Mom's Books (Word Docs)
- Published author with manuscripts in Word format
- Complex formatting: chapters, footnotes, block quotes
- Need: mammoth.js adapter with chapter-aware heading detection
- Pipeline: `Word docs → Billet → markdown → Anvil → searchable library`

### QuoteAI Source Material
- Product catalogs, spec sheets, pricing docs in various formats
- Need: table extraction (pricing), structured data preservation
- Pipeline: `Various docs → Billet → markdown → QuoteAI knowledge base`

---

## Architecture Decisions (Preliminary)

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Separate package from Anvil | Single responsibility — Max (GMPPU AI) validated this |
| 2 | Adapter pattern for formats | Each format has unique challenges; plugin architecture |
| 3 | CLI + library API (like Anvil) | CLI for batch processing, API for programmatic use |
| 4 | Local-first, no API keys | Same philosophy as Anvil v1 |
| 5 | Markdown-only output | Keep it focused — other output formats are a different tool |

---

## Open Questions

- **PDF quality:** How good is pdf-parse for complex layouts? May need pdf.js or a commercial lib for FIA-quality docs.
- **Image extraction:** Do we extract embedded images and reference them in markdown? (v2 probably)
- **Chunking hints:** Should Billet emit hints that help Anvil chunk better? (e.g., "this is a chapter boundary")
- **Incremental processing:** Watch for new/changed files and re-process only what changed?
- **Testing strategy:** How do we test format conversion quality? Visual diff? Gold-standard markdown comparisons?

---

## Ecosystem Fit

```
@claymore-dev/billet  → Format conversion (raw → markdown)
@claymore-dev/anvil   → Semantic search (markdown → embeddings → search)
@claymore-dev/foundry → Documentation platform (search + UI + MCP)
```

Each package does one thing well. Together they're a complete document intelligence pipeline.

---

## Next Steps (When We're Ready)

- [ ] Prototype Word doc adapter (mammoth.js — lowest hanging fruit)
- [ ] Test with Dan's mom's book manuscript
- [ ] Prototype PDF adapter with an FIA regulation doc
- [ ] Evaluate pdf-parse vs pdf.js vs commercial options for complex PDFs
- [ ] Define CLI interface (`billet convert ./input --output ./markdown`)
- [ ] Publish as `@claymore-dev/billet`

---

## Related

- [Anvil Design Doc](../anvil/design.md) — downstream search engine
- [Foundry Design Doc](../foundry/design.md) — doc platform consuming Anvil
- [QuoteAI Design Doc](../quoteai/design.md) — potential consumer of Billet output
