# Usage Patterns

This page is the shortest answer to:

- what object to build
- which function to call
- when to use each API

For a complete runnable example, download the executed notebook:

- [full local SDK walkthrough](notebooks/real-usage-with-mojo.ipynb)
- [batch-search-on-one-loaded-lancedb-slice.ipynb](notebooks/batch-search-on-one-loaded-lancedb-slice.ipynb)
- [lancedb-to-kayak-reranking.ipynb](notebooks/lancedb-to-kayak-reranking.ipynb)

If you already have a vector database and want Kayak to own retrieval, see:

- [Storage + Search](storage-and-search.md)

If you are choosing an encoder first, read:

- [Text Encoders](text-encoders.md)

## Recommendation First

If you just want the recommended public path:

- use `kayak.open_text_retriever(...)` when your app starts from text
- use `kayak.query(...)` plus `kayak.documents(...).pack()` when you already own token vectors
- reuse one loaded or packed `LateIndex` when many queries hit the same slice
- use `kayak.search_batch(...)` for repeated-query throughput on that same index
- use `kayak.search(...)` when you only need top-k hits
- use `kayak.maxsim(...)` only when you actually need the full score vector

Current low-level fast path, as implemented:

- `kayak.search(...)` or `kayak.search_batch(...)`
- `backend=kayak.MOJO_EXACT_CPU_BACKEND`
- packed index layout
- nested query layout

That is why the docs should push users toward index reuse and batch search rather than backend-specific application primitives.

## Choose The API Shape

=== "I Start From Plain Text"

    Use one retriever object:

    - `kayak.open_text_retriever(...)`
    - `retriever.upsert_texts(...)`
    - `retriever.search_text(...)`
    - `retriever.search_text_batch(...)` for repeated query traffic

    Choose this when your application inputs are plain text and you do not want to
    manage encoder and store objects separately.

=== "I Already Have Vectors"

    Use the low-level exact search surface:

    - `kayak.query(...)`
    - `kayak.documents(...).pack()`
    - `kayak.search(...)`
    - `kayak.search_batch(...)` when the same index serves many queries

    Choose this when your application already owns token-level query and
    document vectors.

=== "I Want To Keep My Database"

    Keep the database for persistence and let Kayak materialize the searchable
    slice:

    - `kayak.open_store(...)`
    - `store.load_index(...)`
    - `kayak.search(...)` or `kayak.search_batch(...)`

    Choose this when LanceDB, PgVector, Qdrant, Weaviate, or Chroma is already
    part of your system.

=== "I Need Profiles Or Candidate Stages"

    Use an explicit search plan:

    - `kayak.exact_full_scan_search_plan(...)`
    - `kayak.document_proxy_search_plan(...)`
    - `kayak.search_with_plan(...)`

    Choose this when you need stage accounting, candidate windows, or exact
    reranking after an approximate first stage.

## Plain-Text Ingest Plus Search

Use this when your application starts from text and you want one SDK object for
ingest plus retrieval.

This is not a document-parsing path. If your inputs are PDFs, scans, tables, or
other mixed-layout files, parse or extract them first and pass the resulting text
or token-level vectors into Kayak.

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
- `retriever.upsert_texts(...)` when the source corpus starts as plain text
- `retriever.search_text(...)` when you want the retriever to encode the query and load the store slice
- `retriever.load_index(...)` when you want to reuse one materialized slice across many queries
- omit `backend=...` if the default backend behavior is enough for your application

Default advice:

- use the retriever first for simplicity
- move to explicit `load_index(...)` plus `search_batch(...)` when repeated-query performance matters

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

This is also the current documented fast path for repeated-query public SDK use.

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

If your application should normally use one explicit backend, define it once and reuse it.

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
| Plain-text ingest plus text search | `kayak.open_text_retriever(...)` |
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

- [full local SDK walkthrough](notebooks/real-usage-with-mojo.ipynb)

For vector-database handoff patterns, see:

- [Vector Database Integrations](vector-databases.md)
- [lancedb-to-kayak-reranking.ipynb](notebooks/lancedb-to-kayak-reranking.ipynb)
