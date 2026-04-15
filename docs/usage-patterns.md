# Usage Patterns

This page is the shortest answer to:

- what object to build
- which function to call
- when to use each API

For a complete runnable example, download the executed notebook:

- [real-usage-with-mojo.ipynb](notebooks/real-usage-with-mojo.ipynb)
- [batch-search-on-one-loaded-lancedb-slice.ipynb](notebooks/batch-search-on-one-loaded-lancedb-slice.ipynb)
- [lancedb-to-kayak-reranking.ipynb](notebooks/lancedb-to-kayak-reranking.ipynb)

If you already have a vector database and want Kayak to own retrieval, see:

- [Storage + Search](storage-and-search.md)

If your data already lives in a hosted-engine snapshot and you want one local
Python reuse or scheduling surface around that snapshot, see:

- [Hosted Engine Python](hosted-engine-python.md)

If you are choosing an encoder first, read:

- [Text Encoders](text-encoders.md)

## Text Ingest Plus Search

Use this when your application starts from text and you want one SDK object for
ingest plus retrieval.

```python
import kayak

retriever = kayak.open_text_retriever(
    encoder="colbert",
    store="kayak",
    encoder_kwargs={"model_name": "colbert-ir/colbertv2.0"},
    store_kwargs={"path": "./kayak-index"},
)

retriever.upsert_texts(doc_ids, texts, metadata=metadata_rows)
hits = retriever.search_text(query_text, k=5, where={"tenant": "acme"})
```

Use:

- `kayak.open_text_retriever(...)` for one high-level text workflow object
- `retriever.upsert_texts(...)` when the source corpus starts as raw text
- `retriever.search_text(...)` when you want the retriever to encode the query and load the store slice
- `retriever.load_index(...)` when you want to reuse one materialized slice across many queries
- omit `backend=...` if you want this high-level workflow to prefer Mojo automatically when available

Choose the encoder this way:

- `"colbert"` when you have a ColBERT checkpoint on Hugging Face
- `"callable"` when you already have your own model methods

## Exact Search For One Query

Use this when you already have one query embedding and one packed index.

```python
import kayak

BACKEND = kayak.MOJO_EXACT_CPU_BACKEND

query = kayak.query(query_vectors, text=query_text)
index = kayak.documents(
    doc_ids,
    document_vectors,
    texts=document_texts,
).pack()

hits = kayak.search(query, index, k=5, backend=BACKEND)
```

Use:

- `kayak.query(...)` for one query
- `kayak.documents(...).pack()` for one reusable index
- `kayak.search(...)` when you want top-k hits only

## Exact Scores For All Documents

Use this when you need the full score vector, not only top-k hits.

```python
scores = kayak.maxsim(query, index, backend=BACKEND)
top_hits = scores.topk(5)
```

Use:

- `kayak.maxsim(...)` for one query against all indexed documents
- `LateScores.topk(k)` when you want hits after inspecting scores

## Batch Search Against The Same Index

Use this when the index stays fixed and you want to run multiple queries.

```python
batch = kayak.query_batch([query_a_vectors, query_b_vectors, query_c_vectors])

hits_by_query = kayak.search_batch(
    batch,
    index,
    k=5,
    backend=BACKEND,
)
```

Use:

- `kayak.query_batch(...)` to keep ragged query vector counts explicit
- `kayak.search_batch(...)` for top-k hits per query
- `kayak.maxsim_batch(...)` if you need full score vectors per query

If you are already using the high-level retriever and your queries still start
as text, use:

```python
hits_by_query = retriever.search_text_batch(
    [query_a_text, query_b_text, query_c_text],
    k=5,
    where={"tenant": "acme"},
)
```

- `retriever.search_text_batch(...)` when you want one store load plus one batch search
- `retriever.search_query_batch(...)` when you already built a `LateQueryBatch`

This is the right public SDK shape when many queries target the same already
loaded slice. It is different from generic concurrent use of one store object.

## Exact Baseline With Stage Profiles

Use this when you want exact retrieval plus explicit stage accounting.

```python
plan = kayak.exact_full_scan_search_plan(final_k=5)

result = kayak.search_with_plan(
    query,
    index,
    plan,
    backend=BACKEND,
)
```

Use:

- `kayak.exact_full_scan_search_plan(...)` as the exact baseline
- `kayak.search_with_plan(...)` when you want profiles, candidate-stage output, and stage metadata

## Approximate Candidate Stage Plus Exact Rerank

Use this when you want a smaller candidate window before exact late interaction.

```python
plan = kayak.document_proxy_search_plan(
    final_k=5,
    candidate_k=100,
    query_vector_budget=32,
    document_vector_budget=64,
)

result = kayak.search_with_plan(
    query,
    index,
    plan,
    backend=BACKEND,
)
```

Use:

- `candidate_k` to control candidate-window size
- `query_vector_budget` and `document_vector_budget` to control proxy-stage cost
- `result.candidate_stage.profile` and `result.stage2` to inspect the actual work done

## Vector DB For Storage, Kayak For Search

Use this when your database is the durable store and Kayak is the search
engine.

```python
rows = vector_db.fetch_all()

index = kayak.documents(
    [row["doc_id"] for row in rows],
    [row["vector"] for row in rows],
    texts=[row["text"] for row in rows],
).pack()

hits = kayak.search(query, index, k=10, backend=BACKEND)
```

Use:

- the vector database for saving
- `kayak.open_store(...)` when you want Kayak to materialize that database into one searchable slice directly
- `kayak.documents(...).pack()` as the reusable in-process search index
- `kayak.search(...)` or `kayak.search_with_plan(...)` for query-time retrieval

For the benchmark-backed version of this pattern, see:

- [Storage + Search](storage-and-search.md)

## Many Same-Process Callers Against One Fixed Snapshot

Use this when your data already lives in a hosted-engine service root and many
callers in one Python process need the same pinned snapshot.

```python
from kayak_engine import (
    PreparedExactSearchSchedulerConfig,
    prepare_exact_search_scheduler,
)

scheduler = prepare_exact_search_scheduler(
    service_root="./.state/kayak-engine",
    collection_id="news",
    tenant_id="tenant-a",
    namespace_id="search",
    snapshot_id="snapshot-0001",
    config=PreparedExactSearchSchedulerConfig(
        worker_count=4,
        max_batch_size=32,
        max_batch_wait_ms=2,
    ),
)
```

Use:

- `kayak.search_batch(...)` when you already loaded one local `LateIndex`
- `kayak_engine.prepare_exact_search_scheduler(...)` when many callers share one fixed hosted snapshot
- [Hosted Engine Python](hosted-engine-python.md) for the full scheduler contract

## Clause-Text Verification

Use this only when you want the text-family verifier.

You must attach:

- `text=` to the query
- `texts=` to the documents or index

```python
query = kayak.query(query_vectors, text=query_text)
index = kayak.documents(
    doc_ids,
    document_vectors,
    texts=document_texts,
).pack()

plan = kayak.exact_full_scan_search_plan(
    final_k=5,
    candidate_k=20,
    stage3_verifier=kayak.clause_text_stage3_verifier_operator(),
)

result = kayak.search_with_plan(
    query,
    index,
    plan,
    backend=BACKEND,
)
```

## Explicit 128-Dim Layouts

Use this when your embeddings are 128-dimensional and you want the layout choice
to be explicit in code.

```python
flat_query = query.to_layout("flat_dim128")
hybrid_index = index.to_layout("hybrid_flat_dim128")

scores = kayak.maxsim(
    flat_query,
    hybrid_index,
    backend=BACKEND,
)
```

## Backend Selection

If your application should normally use Mojo, define it once and reuse it.

```python
BACKEND = kayak.MOJO_EXACT_CPU_BACKEND
```

If you want to inspect what the runtime can actually use:

```python
print(kayak.available_backends())
print(kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND))
```

## Minimal API Map

| Task | API |
| --- | --- |
| Text ingest plus text search | `kayak.open_text_retriever(...)` |
| One query, top-k hits | `kayak.search(...)` |
| One query, all exact scores | `kayak.maxsim(...)` |
| Many queries, top-k hits | `kayak.search_batch(...)` |
| Many queries, all exact scores | `kayak.maxsim_batch(...)` |
| Exact baseline with stage metadata | `kayak.exact_full_scan_search_plan(...)` + `kayak.search_with_plan(...)` |
| Approximate candidates + exact rerank | `kayak.document_proxy_search_plan(...)` + `kayak.search_with_plan(...)` |
| Text-family verification | `kayak.clause_text_stage3_verifier_operator()` |
| 128-dim explicit layouts | `query.to_layout(...)`, `index.to_layout(...)` |

## Notebook

The notebook uses a real text corpus, encodes it to ColBERT-style 128-dimensional
vectors on CPU, builds a Kayak index, and runs:

- exact search
- flat/hybrid layout scoring
- approximate candidate generation plus exact reranking
- batch search
- clause-text verification

Notebook:

- [real-usage-with-mojo.ipynb](notebooks/real-usage-with-mojo.ipynb)

For vector-database handoff patterns, see:

- [Vector Database Integrations](vector-databases.md)
- [lancedb-to-kayak-reranking.ipynb](notebooks/lancedb-to-kayak-reranking.ipynb)
