---
hide:
  - toc
---

<div class="kayak-hero kayak-hero--compact" markdown>
<div class="kayak-hero__main" markdown>

<p class="kayak-eyebrow">Choose by workflow</p>

# Pick the entry point that matches what you already have

<p class="kayak-lead">
Most users do not start from `LateIndex`, search plans, or storage protocols.
They start from a practical situation: text, vectors, an existing database, or
repeated-query traffic.
</p>

<div class="kayak-action-row" markdown>

[Installation](installation.md){ .md-button .md-button--primary }
[Quickstart](quickstart.md){ .md-button }
[Examples](examples.md){ .md-button }

</div>

</div>
<aside class="kayak-hero__aside" markdown>

<ul class="kayak-link-list">
  <li><strong>Fastest route</strong> Installation → Quickstart → Usage Patterns</li>
  <li><strong>Database-first route</strong> Storage + Search → Vector Databases</li>
  <li><strong>Hosted serving route</strong> Hosted Engine Python</li>
</ul>

</aside>
</div>

## Choose Your Starting Situation

<div class="kayak-card-grid" markdown>

<section class="kayak-card kayak-card--accent" markdown>
### I want one working search first

Use this when the immediate goal is a verified local search on the Mojo-backed
exact path.

[Installation](installation.md)
[Quickstart](quickstart.md)
[real-usage-with-mojo.ipynb](notebooks/real-usage-with-mojo.ipynb)
</section>

<section class="kayak-card" markdown>
### I start from raw text

Use this when your corpus begins as strings and you want Kayak to own encoding
plus search behind one Python retriever workflow.

[Text Encoders](text-encoders.md)
[Usage Patterns](usage-patterns.md)
</section>

<section class="kayak-card" markdown>
### I already have token vectors

Use this when you already own token-level query and document vectors and want
to search directly.

[Quickstart](quickstart.md)
[Python API](api.md)
</section>

<section class="kayak-card" markdown>
### I already have a vector database

Use this when LanceDB, PgVector, Qdrant, Weaviate, or Chroma is already your
durable store and you want Kayak to take over search.

[Storage + Search](storage-and-search.md)
[Vector Databases](vector-databases.md)
</section>

<section class="kayak-card" markdown>
### I need repeated-query speed

Use this when the loaded slice stays fixed for a while and query traffic is the
moving part.

[Usage Patterns](usage-patterns.md)
[batch-search-on-one-loaded-lancedb-slice.ipynb](notebooks/batch-search-on-one-loaded-lancedb-slice.ipynb)
</section>

<section class="kayak-card" markdown>
### I need one same-snapshot runtime for many callers

Use this when one hosted snapshot serves many requests inside one Python
process and you need prepared runtime lanes.

[Hosted Engine Python](hosted-engine-python.md)
</section>

</div>

## The Main API You Probably Want

| If your situation is... | The first API to look at is... |
| --- | --- |
| raw text in, search results out | `kayak.open_text_retriever(...)` |
| precomputed vectors in, exact search out | `kayak.query(...)`, `kayak.documents(...).pack()`, `kayak.search(...)` |
| database keeps persistence | `kayak.open_store(...)` plus `store.load_index(...)` |
| repeated queries on one fixed slice | `store.load_index(...)` once plus `kayak.search_batch(...)` |
| same-snapshot serving in one process | `prepare_exact_search_runtime(...)` |

## The Shortest Safe Reading Order

<div class="kayak-card-grid" markdown>

<section class="kayak-card" markdown>
### Install and verify

Confirm that Kayak can see a usable `mojo` CLI and that the exact backend is
actually available.

[Installation](installation.md)
</section>

<section class="kayak-card" markdown>
### Run one working example

Do this before tuning adapters, layouts, or plans.

[Quickstart](quickstart.md)
</section>

<section class="kayak-card" markdown>
### Then pick the long-term shape

This is where you decide between retriever workflows, loaded-slice reuse, and
database handoff.

[Usage Patterns](usage-patterns.md)
</section>

</div>

## What People Usually Mean

| If you are thinking... | Open... |
| --- | --- |
| “How do I make it use Mojo?” | [Installation](installation.md) |
| “Can I just pass a Hugging Face model?” | [Text Encoders](text-encoders.md) |
| “Can I keep Postgres, Qdrant, LanceDB, Weaviate, or Chroma?” | [Storage + Search](storage-and-search.md) |
| “What is the fastest repeated-query path?” | [Usage Patterns](usage-patterns.md) |
| “I need runnable artifacts, not prose.” | [Examples](examples.md) |
