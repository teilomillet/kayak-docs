# Where Kayak Fits

Kayak is the late-interaction search layer.

It is not a full document-intelligence or RAG-ingestion platform.

## The Boundary

Use Kayak after documents have become either:

- token-level query and document vectors
- plain text that you pass through an explicit encoder
- a materialized search slice loaded from your existing storage system

Do not expect Kayak to own:

- OCR
- PDF layout recovery
- table extraction
- handwritten annotation handling
- application answer generation

Those are real systems. Keep them on your side or connect Kayak to a dedicated
parser, extractor, or application stack.

## The Pipeline Shape

```text
documents, PDFs, or databases
        |
        v
parser / OCR / extractor / application ETL
        |
        v
plain text or token-level vectors
        |
        v
encoder, if vectors are not already materialized
        |
        v
Kayak: late-interaction index, search plan, exact rerank, explain data
        |
        v
application workflow, answer generation, or user interface
```

Kayak owns the middle search layer:

- explicit late-interaction objects
- exact MaxSim scoring
- candidate stages plus exact reranking
- vector-count and layout visibility
- database handoff and search-slice materialization
- repeated-query and hosted-snapshot reuse

## Why The Boundary Is Narrow

Late interaction has a different cost model from one-vector dense retrieval.

Search quality, memory, and latency depend on:

- query vector count
- document vector count
- candidate window size
- layout
- backend
- whether stage 1 actually reaches the documents that exact reranking needs

Kayak keeps those details visible so you can measure and tune them directly.
That would be harder if the product boundary hid retrieval inside a broad
"upload documents and ask questions" pipeline.

## What To Optimize In Kayak

Optimize Kayak when the problem is:

- exact late-interaction throughput
- candidate recall per unit cost
- bytes per vector or bytes per document
- repeated-query serving over one fixed search slice
- explicit stage profiles and explainability
- keeping an existing database while Kayak owns the search step

Use other systems when the problem is:

- scanned document cleanup
- layout-aware document parsing
- structured field extraction
- chunk authoring for non-late-interaction retrievers
- answer synthesis and workflow automation

## Plain Text Support

`kayak.open_text_retriever(...)` is a convenience path for plain text plus an
explicit encoder.

It does not turn Kayak into an OCR or extraction engine.

Use it when your application already has text strings and you want one object
to wire together:

- encoder
- store
- search

If your source data is PDFs, scans, spreadsheets, emails, or mixed-layout
documents, run the appropriate parser or extractor first. Then pass the
resulting text or token-level vectors into Kayak.
