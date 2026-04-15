<div class="kayak-hero kayak-hero--compact" markdown>
<div class="kayak-hero__main" markdown>

<p class="kayak-eyebrow">Runnable artifacts</p>

# Open the notebook or script that matches the job

<p class="kayak-lead">
This page is the shortest route to real, runnable Kayak examples. Use it when
you want to inspect working code and measured outputs instead of reading
conceptual pages first.
</p>

<div class="kayak-action-row" markdown>

[real-usage-with-mojo.ipynb](notebooks/real-usage-with-mojo.ipynb){ .md-button .md-button--primary }
[batch-search-on-one-loaded-lancedb-slice.ipynb](notebooks/batch-search-on-one-loaded-lancedb-slice.ipynb){ .md-button }

</div>

</div>
<aside class="kayak-hero__aside" markdown>

<ul class="kayak-link-list">
  <li><strong>Best general walkthrough</strong> <code>real-usage-with-mojo.ipynb</code></li>
  <li><strong>Best repeated-query example</strong> <code>batch-search-on-one-loaded-lancedb-slice.ipynb</code></li>
  <li><strong>Best integration set</strong> LanceDB, PgVector, Qdrant, Weaviate, and Chroma notebooks</li>
</ul>

</aside>
</div>

## Choose An Example By Goal

<div class="kayak-card-grid" markdown>

<section class="kayak-card kayak-card--accent" markdown>
### End-to-end SDK walkthrough

Use this when you want one broad public example from encoding to exact search.

[Open notebook](notebooks/real-usage-with-mojo.ipynb)
</section>

<section class="kayak-card" markdown>
### Repeated-query speed

Use this when you want the loaded-slice plus batch-search path.

[Open notebook](notebooks/batch-search-on-one-loaded-lancedb-slice.ipynb)
</section>

<section class="kayak-card" markdown>
### Keep the existing database

Use these when you want the “database for storage, Kayak for search” story.

[LanceDB](notebooks/lancedb-to-kayak-reranking.ipynb)
[PgVector](notebooks/pgvector-to-kayak-exact-search.ipynb)
[Qdrant](notebooks/qdrant-to-kayak-reranking.ipynb)
[Weaviate](notebooks/weaviate-to-kayak-reranking.ipynb)
[Chroma](notebooks/chromadb-to-kayak-reranking.ipynb)
</section>

<section class="kayak-card" markdown>
### Small script examples

Use these when you want the smallest possible repository examples instead of a
notebook.

<code>python/examples/query_batch.py</code>
<code>python/examples/text_retriever_workflow.py</code>
<code>python/examples/pgvector_store.py</code>
</section>

</div>

## Executed Notebooks

| Example | Best for |
| --- | --- |
| [real-usage-with-mojo.ipynb](notebooks/real-usage-with-mojo.ipynb) | full local SDK walkthrough from text encoding to exact search, plans, and batch search |
| [batch-search-on-one-loaded-lancedb-slice.ipynb](notebooks/batch-search-on-one-loaded-lancedb-slice.ipynb) | repeated-query workflows on one loaded exact slice |
| [lancedb-to-kayak-reranking.ipynb](notebooks/lancedb-to-kayak-reranking.ipynb) | LanceDB persistence with Kayak reranking and exact search |
| [pgvector-to-kayak-exact-search.ipynb](notebooks/pgvector-to-kayak-exact-search.ipynb) | Postgres plus pgvector as durable storage with exact slice materialization |
| [qdrant-to-kayak-reranking.ipynb](notebooks/qdrant-to-kayak-reranking.ipynb) | Qdrant candidate retrieval with Kayak exact reranking |
| [weaviate-to-kayak-reranking.ipynb](notebooks/weaviate-to-kayak-reranking.ipynb) | Weaviate storage with Kayak exact search handoff |
| [chromadb-to-kayak-reranking.ipynb](notebooks/chromadb-to-kayak-reranking.ipynb) | Chroma compatibility path with exact Kayak reranking |

## Repository Scripts

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

## Open The Companion Page You Need

| If you want to... | Open... |
| --- | --- |
| choose by situation instead of artifact type | [Choose Your Path](start-here.md) |
| get the shortest Mojo-first script | [Quickstart](quickstart.md) |
| choose between retrievers, loaded slices, and plans | [Usage Patterns](usage-patterns.md) |
| understand the database handoff model | [Storage + Search](storage-and-search.md) |
| inspect adapter-specific storage facts | [Vector Databases](vector-databases.md) |
