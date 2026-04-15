# Examples

This page is the shortest map to real, runnable Kayak examples.

Use these when you do not want another conceptual page and just want to see the
actual shape of the API in practice.

## Choose An Example By Goal

=== "I Need One End-To-End Demo"

    Open:

    - [real-usage-with-mojo.ipynb](notebooks/real-usage-with-mojo.ipynb)

=== "I Need To Show Repeated-Query Speed"

    Open:

    - [batch-search-on-one-loaded-lancedb-slice.ipynb](notebooks/batch-search-on-one-loaded-lancedb-slice.ipynb)

=== "I Need To Show We Can Keep The Existing Database"

    Open:

    - [lancedb-to-kayak-reranking.ipynb](notebooks/lancedb-to-kayak-reranking.ipynb)
    - [pgvector-to-kayak-exact-search.ipynb](notebooks/pgvector-to-kayak-exact-search.ipynb)
    - [qdrant-to-kayak-reranking.ipynb](notebooks/qdrant-to-kayak-reranking.ipynb)
    - [weaviate-to-kayak-reranking.ipynb](notebooks/weaviate-to-kayak-reranking.ipynb)
    - [chromadb-to-kayak-reranking.ipynb](notebooks/chromadb-to-kayak-reranking.ipynb)

=== "I Need The Smallest Script, Not A Notebook"

    Open:

    - `python/examples/query_batch.py`
    - `python/examples/text_retriever_workflow.py`
    - `python/examples/pgvector_store.py`

## Start Here

If you want one broad public SDK walkthrough:

- [real-usage-with-mojo.ipynb](notebooks/real-usage-with-mojo.ipynb)

If you want many queries against the same loaded slice:

- [batch-search-on-one-loaded-lancedb-slice.ipynb](notebooks/batch-search-on-one-loaded-lancedb-slice.ipynb)

If you want the shortest raw batch API script:

- `python/examples/query_batch.py`

## Executed Notebooks

<div class="grid cards" markdown>

- __Public SDK walkthrough__

  Full local flow from text encoding to exact search, plans, batch search, and
  verification.

  [Open notebook](notebooks/real-usage-with-mojo.ipynb)

- __Batch search on one loaded slice__

  Shows the supported repeated-query path: `load_index(...)` once, then reuse
  that `LateIndex` with `search_batch(...)`.

  [Open notebook](notebooks/batch-search-on-one-loaded-lancedb-slice.ipynb)

- __LanceDB handoff__

  Keep LanceDB for storage, use Kayak for exact reranking and surfacing.

  [Open notebook](notebooks/lancedb-to-kayak-reranking.ipynb)

- __PgVector exact slice__

  Keep Postgres plus `pgvector` as the durable store and materialize one exact
  filtered slice into Kayak.

  [Open notebook](notebooks/pgvector-to-kayak-exact-search.ipynb)

- __Qdrant handoff__

  Store native multivectors in Qdrant, then rerank candidate windows in Kayak.

  [Open notebook](notebooks/qdrant-to-kayak-reranking.ipynb)

- __Weaviate handoff__

  Use a named multivector in Weaviate, then move exact scoring into Kayak.

  [Open notebook](notebooks/weaviate-to-kayak-reranking.ipynb)

- __Chroma handoff__

  Use dense stage-1 retrieval in Chroma and exact token-level reranking in
  Kayak.

  [Open notebook](notebooks/chromadb-to-kayak-reranking.ipynb)

</div>

## Python Scripts

These are smaller script examples in the repository.

| Task | File |
| --- | --- |
| ColBERT from Hugging Face | `python/examples/colbert_hf_encoder.py` |
| Bring your own model wrapper | `python/examples/byo_model_encoder.py` |
| High-level retriever workflow | `python/examples/text_retriever_workflow.py` |
| Encoder plus store workflow | `python/examples/encoder_store_workflow.py` |
| Raw query batch API | `python/examples/query_batch.py` |
| LanceDB store adapter | `python/examples/lancedb_store.py` |
| PgVector store adapter | `python/examples/pgvector_store.py` |
| Qdrant store adapter | `python/examples/qdrant_store.py` |
| Weaviate store adapter | `python/examples/weaviate_store.py` |
| Chroma store adapter | `python/examples/chromadb_store.py` |

## Which One To Open

Use:

- [Start Here](start-here.md) when you are still choosing based on your situation, not the API names
- [Quickstart](quickstart.md) when you want the shortest Mojo-first script
- [Usage Patterns](usage-patterns.md) when you want help choosing an API
- [Storage + Search](storage-and-search.md) when your database already exists
- [Vector Databases](vector-databases.md) when you want adapter-specific facts
- [API Reference](api.md) when you already know the shape and want signatures
