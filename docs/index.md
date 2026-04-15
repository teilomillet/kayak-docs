---
hide:
  - toc
---

<div class="kayak-home-header" markdown>

<p class="kayak-eyebrow">Kayak Documentation</p>

# Exact late-interaction retrieval for Python

<p class="kayak-lead">
Kayak gives you an explicit Python surface for exact late-interaction search,
with a real Mojo-backed exact path and a practical way to keep your current
vector database when that is already the durable system of record.
</p>

<div class="kayak-action-row" markdown>

[Get started](installation.md){ .md-button .md-button--primary }
[Quickstart](quickstart.md){ .md-button }
[Python API](api.md){ .md-button }

</div>

</div>

<div class="kayak-home-link-grid">
  <a class="kayak-home-link" href="installation.md">
    <strong>Install and verify</strong>
    <span>Set up Kayak in one environment where a usable <code>mojo</code> CLI is visible and verify the exact backend before you do anything else.</span>
  </a>
  <a class="kayak-home-link" href="quickstart.md">
    <strong>Run one exact search</strong>
    <span>Use the shortest verified local example first, then move to the longer workflow you actually need.</span>
  </a>
  <a class="kayak-home-link" href="text-encoders.md">
    <strong>Start from text or your own model</strong>
    <span>Use a ColBERT checkpoint or wrap your own model and keep the encoding surface explicit.</span>
  </a>
  <a class="kayak-home-link" href="storage-and-search.md">
    <strong>Keep the existing database</strong>
    <span>Use the database for storage and filtering, then let Kayak own the exact searchable slice and the search step.</span>
  </a>
</div>

## Common Workflows

<div class="kayak-card-grid" markdown>

<section class="kayak-card kayak-card--accent" markdown>
### Local text retrieval

Use this when your input starts as raw text and you want one ergonomic object
for ingest, indexing, and search.

- [Text Encoders](text-encoders.md)
- [Usage Patterns](usage-patterns.md)
- `kayak.open_text_retriever(...)`
</section>

<section class="kayak-card" markdown>
### Exact search from token vectors

Use this when you already own token-level vectors and want direct exact search
without the retriever layer.

- [Quickstart](quickstart.md)
- [Python API](api.md)
- `kayak.query(...)`, `kayak.documents(...).pack()`, `kayak.search(...)`
</section>

<section class="kayak-card" markdown>
### Database handoff

Use this when LanceDB, PgVector, Qdrant, Weaviate, or Chroma already exists in
your system and you want Kayak to take over retrieval.

- [Storage + Search](storage-and-search.md)
- [Vector Databases](vector-databases.md)
- `kayak.open_store(...)`, `store.load_index(...)`
</section>

<section class="kayak-card" markdown>
### Repeated-query serving

Use this when the slice stays fixed for a while and the query traffic is the
part that changes.

- [Usage Patterns](usage-patterns.md)
- [Hosted Engine Python](hosted-engine-python.md)
- `kayak.search_batch(...)`
</section>

</div>

## What Kayak Keeps Explicit

<div class="kayak-card-grid" markdown>

<section class="kayak-card" markdown>
### Search semantics

- query vector count
- document vector count
- index layout
- backend choice
- candidate window size
</section>

<section class="kayak-card" markdown>
### Python ergonomics

- `kayak.help()` for discoverability
- `kayak.doctor()` for environment diagnostics
- retriever and session APIs for repeated-query workloads
</section>

<section class="kayak-card" markdown>
### Storage boundary

- keep the existing vector database when it already owns persistence
- let Kayak materialize one exact searchable slice
- measure the search path you actually plan to deploy
</section>

</div>

<figure class="kayak-figure">
  <img src="assets/overview-flow.svg" alt="Overview of the public Kayak flow from text or vectors into exact search, with optional vector database storage and loaded-slice reuse.">
  <figcaption>Practical mental model: text or vectors in, exact searchable slice out, optional database persistence on the side.</figcaption>
</figure>

## Start With The Right Page

| If you want to... | Open... |
| --- | --- |
| choose by your situation instead of by API name | [Choose Your Path](start-here.md) |
| install Kayak and make Mojo available correctly | [Installation](installation.md) |
| run the shortest possible exact search | [Quickstart](quickstart.md) |
| pass a Hugging Face ColBERT checkpoint or your own model | [Text Encoders](text-encoders.md) |
| keep your existing vector database | [Storage + Search](storage-and-search.md) |
| compare adapter-specific behavior | [Vector Databases](vector-databases.md) |
| open runnable notebooks and scripts | [Examples](examples.md) |
| inspect the stable public signatures | [Python API](api.md) |

## Start With These Three

1. [Installation](installation.md)
2. [Quickstart](quickstart.md)
3. [Usage Patterns](usage-patterns.md)

That gets a new user from environment setup to one working search to the right
long-term API shape without forcing them through the full reference first.
