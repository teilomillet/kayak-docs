# Start Here

<div class="kayak-hero kayak-hero--compact" markdown>

<p class="kayak-eyebrow">Choose by situation, not by API name</p>

# Start Here

<p class="kayak-lead">
This page is for the point where a user knows what they have, but not yet what
Kayak calls it.
</p>

[Quickstart](quickstart.md){ .md-button .md-button--primary }
[Examples](examples.md){ .md-button }
[API Reference](api.md){ .md-button }

</div>

This page is for the moment before a user knows the Kayak vocabulary.

Most people do not begin by asking for a `LateIndex` or a search plan. They
begin with one of these situations:

- "I want one working search first."
- "I already have text."
- "I already have vectors."
- "I already have a vector database."
- "I need repeated-query speed."
- "I need one same-snapshot runtime for many callers."

## Fast Mental Model

<div class="grid cards" markdown>

- __I have text__

  Use one retriever object and let Kayak own encoding plus search.

  [Open text path](text-encoders.md)

- __I have vectors__

  Build one query, one packed index, and search directly.

  [Open vector path](quickstart.md)

- __I have a vector DB__

  Keep the database for storage and let Kayak own retrieval.

  [Open storage path](storage-and-search.md)

- __I have repeated query traffic__

  Load one slice once and reuse it with batch search.

  [Open repeated-query path](usage-patterns.md)

</div>

## Choose Your Situation

=== "I Want One Working Search"

    Use this path when your real goal is: "show me a working local Kayak search
    on Mojo as quickly as possible."

    Read in this order:

    1. [Installation](installation.md)
    2. [Quickstart](quickstart.md)
    3. [real-usage-with-mojo.ipynb](notebooks/real-usage-with-mojo.ipynb)

    Stop here first if you do not want to think about encoders, stores, or
    plans yet.

=== "I Start From Text"

    Use this path when your corpus starts as raw text and you want one SDK
    object for ingest plus search.

    Read in this order:

    1. [Text Encoders](text-encoders.md)
    2. [Usage Patterns](usage-patterns.md)
    3. `python/examples/text_retriever_workflow.py`

    Main API:

    - `kayak.open_text_retriever(...)`
    - `retriever.upsert_texts(...)`
    - `retriever.search_text(...)`

=== "I Already Have Vectors"

    Use this path when you already own token-level query and document vectors.

    Read in this order:

    1. [Quickstart](quickstart.md)
    2. [API Reference](api.md)
    3. `python/examples/query_batch.py` if the same index serves many queries

    Main API:

    - `kayak.query(...)`
    - `kayak.documents(...).pack()`
    - `kayak.search(...)`
    - `kayak.search_batch(...)`

=== "I Already Have A Vector Database"

    Use this path when LanceDB, PgVector, Qdrant, Weaviate, or Chroma already
    exists in your system and you want to keep it for persistence.

    Read in this order:

    1. [Storage + Search](storage-and-search.md)
    2. [Vector Databases](vector-databases.md)
    3. The executed notebook for your database

    The mental model is:

    - keep the database for storage
    - let Kayak materialize the exact slice
    - let Kayak own search and surfacing

=== "I Need Repeated-Query Speed"

    Use this path when the data stays fixed for a while and query traffic is the
    thing changing.

    Read in this order:

    1. [Usage Patterns](usage-patterns.md)
    2. [Using the Mojo Backend](mojo-backend.md)
    3. [batch-search-on-one-loaded-lancedb-slice.ipynb](notebooks/batch-search-on-one-loaded-lancedb-slice.ipynb)

    Main pattern:

    - `load_index(...)` once
    - reuse the same `LateIndex`
    - use `search_batch(...)` when queries arrive together

=== "I Need Same-Snapshot Multi-Caller Serving"

    Use this path when one hosted-engine snapshot stays fixed and many callers
    in one Python process need to hit it.

    Read in this order:

    1. [Hosted Engine Python](hosted-engine-python.md)

    Main API:

    - `prepare_exact_search_session(...)` for one caller
    - `prepare_exact_search_runtime(...)` for many callers
    - `concurrency_lane_count` when you need more than one independent lane

## What Users Usually Mean

| If you are thinking... | You probably want... |
| --- | --- |
| "How do I make it use Mojo?" | [Installation](installation.md) and [Using the Mojo Backend](mojo-backend.md) |
| "Can I just pass my Hugging Face model?" | [Text Encoders](text-encoders.md) |
| "Can I keep Qdrant, PgVector, LanceDB, Weaviate, or Chroma?" | [Storage + Search](storage-and-search.md) |
| "What is the fastest supported repeated-query path?" | [Usage Patterns](usage-patterns.md) and [batch-search-on-one-loaded-lancedb-slice.ipynb](notebooks/batch-search-on-one-loaded-lancedb-slice.ipynb) |
| "Do I need Pixi?" | [Installation](installation.md) |
| "I need a real notebook, not docs." | [Examples](examples.md) |

## If You Only Read Three Pages

For most new users, the shortest useful sequence is:

1. [Installation](installation.md)
2. [Quickstart](quickstart.md)
3. [Usage Patterns](usage-patterns.md)

That gets a user from install to one working search to the right API shape
without forcing them to read the full reference first.
