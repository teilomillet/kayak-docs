# API Reference

All public exports come from:

```python
import kayak
```

This page focuses on the stable Python surface and the backend contract.

Hosted-engine Python note:

- this API page covers `import kayak`
- the hosted-engine Python surface lives under `import kayak_engine`
- for pinned hosted snapshots and the local exact-search scheduler, see
  [Hosted Engine Python](hosted-engine-python.md)

## Choose The API Section

=== "I Need The Main Entry Point"

    Start with:

    - `kayak.open_text_retriever(...)` for one high-level text workflow
    - `kayak.open_encoder(...)` if you need encoder control
    - `kayak.open_store(...)` if you need persistence control

=== "I Already Have Vectors"

    Start with:

    - `kayak.query(...)`
    - `kayak.documents(...).pack()`
    - `kayak.search(...)`
    - `kayak.search_batch(...)`

=== "I Need A Database Adapter"

    Jump to:

    - [Stores](#stores)
    - [Storage + Search](storage-and-search.md)
    - [Vector Databases](vector-databases.md)

=== "I Need Plans Or Candidate Stages"

    Jump to:

    - [Candidate Generation And Plans](#candidate-generation-and-plans)
    - [Search Plans](search-plans.md)

=== "I Need Hosted Snapshot Reuse"

    This is the wrong page.

    Use:

    - [Hosted Engine Python](hosted-engine-python.md)

## Constructors

### `kayak.open_encoder(kind, **kwargs)`

Open a public text encoder from the stable factory.

```python
encoder = kayak.open_encoder(
    "callable",
    query_encoder=my_query_encoder,
    document_encoder=my_document_encoder,
)
```

Built-in kinds:

- `"callable"`
- `"colbert"`

Use:

- `"colbert"` when the model is a ColBERT checkpoint and `model_name` is the
  Hugging Face repo id
- `"callable"` when you already have query/document methods that emit token
  vectors

### `kayak.open_store(kind, **kwargs)`

Open a public persistence adapter from the stable factory.

```python
store = kayak.open_store("kayak", path="./kayak-index")
```

Built-in kinds:

- `"memory"`
- `"kayak"` and `"directory"`
- `"lancedb"`
- `"pgvector"` with aliases `"postgres"` and `"postgresql"`
- `"qdrant"`
- `"weaviate"`
- `"chromadb"` and `"chroma"`

### `kayak.open_text_retriever(...)`

Open one high-level text retriever that composes:

- one encoder
- one store
- one default backend

```python
retriever = kayak.open_text_retriever(
    encoder="callable",
    store="memory",
    encoder_kwargs={
        "query_encoder": my_query_encoder,
        "document_encoder": my_document_encoder,
    },
)
```

The `encoder` and `store` arguments can be either:

- a registered factory kind like `"colbert"` or `"kayak"`
- an already-constructed encoder or store object

Backend policy for this high-level constructor:

- prefers `kayak.MOJO_EXACT_CPU_BACKEND` when Mojo is available in the active environment
- falls back to `kayak.NUMPY_REFERENCE_BACKEND` otherwise
- accepts `backend=...` when you want to override the default explicitly

### `kayak.query(token_vectors, *, text=None)`

Build a `LateQuery` from a 2D token-vector matrix.

Accepted inputs are objects that can be converted to a 2D array-like matrix,
including NumPy arrays and torch tensors.

```python
query = kayak.query(vectors)
query_with_text = kayak.query(vectors, text="founded in 1984 in a church")
```

### `kayak.query_batch(token_vectors)`

Build a `LateQueryBatch` from a sequence of query matrices.

Each query can have a different number of vectors. All queries must share the
same vector dimension.

```python
batch = kayak.query_batch([query_a_vectors, query_b_vectors])
```

### `kayak.documents(doc_ids, token_vectors, *, texts=None)`

Build `LateDocuments` from:

- document ids
- one token-vector matrix per document
- optional document texts

```python
documents = kayak.documents(
    ["doc-a", "doc-b"],
    [doc_a_vectors, doc_b_vectors],
    texts=["text a", "text b"],
)
```

## Layout Constructors

These are direct constructors for already-materialized layouts.

### `kayak.packed_index(doc_ids, doc_offsets, token_vectors, *, doc_texts=None)`

Build a `LateIndex` directly in packed layout.

Use this only when you already own packed storage fields.

### `kayak.hybrid_flat_dim128_index(doc_ids, doc_offsets, token_values, *, doc_texts=None)`

Build a `LateIndex` directly in `hybrid_flat_dim128` layout.

Requires a 128-dimensional flattened token-value buffer.

### `kayak.flat_query_dim128(token_values, *, text=None)`

Build a `LateQuery` directly in `flat_dim128` layout from flat values.

Requires 128-dimensional vectors.

## Object Methods

### `LateDocuments.pack()`

Pack `LateDocuments` into a searchable `LateIndex`.

```python
index = documents.pack()
```

## Text Encoders

### `kayak.CallableLateTextEncoder(query_encoder, document_encoder)`

Wrap user-provided Python callables behind Kayak's text encoder contract.

Those callables can come from plain functions or bound model methods.
They are called one text at a time by the public SDK surface.

### `kayak.ColBERTTextEncoder(model_name='colbert-ir/colbertv2.0', *, checkpoint=None, gpus=0)`

Encode text with a ColBERT checkpoint into `LateQuery` and `LateDocuments`.

`model_name` is the Hugging Face repo id for the ColBERT checkpoint.
The public encoder path encodes one query or document text at a time.

### `kayak.register_encoder(kind, factory, *, replace=False)`

Register a custom encoder factory behind `open_encoder(...)`.

## Stores

All public stores support:

- `close()`
- `with ... as store:`
- `load_index(...)` to materialize one exact `LateIndex`

### `kayak.MemoryLateStore()`

Keep late-interaction documents in memory and materialize `LateIndex` objects on demand.

### `kayak.DirectoryLateStore(path)`

Persist one local packed Kayak snapshot on disk and materialize it back into `LateIndex`.

### `kayak.LanceDBLateStore(path, *, table_name='late_documents')`

Persist documents in LanceDB row storage and materialize `LateIndex` objects from the table.

`where=` is applied exactly, but after Arrow materialization in the current public adapter.

### `kayak.PgVectorLateStore(dsn=None, *, connection=None, table_name='late_documents', schema_name='public', ensure_extension=True)`

Persist documents in Postgres with the `pgvector` extension and materialize
exact `LateIndex` objects from stored rows.

`dsn` and `connection` are mutually exclusive. Simple scalar `where=` filters
are pushed into Postgres through JSONB containment before load, and Kayak still
applies the exact filter after fetch.

### `kayak.QdrantLateStore(path=None, *, client=None, collection_name='late_documents')`

Persist documents in a Qdrant collection with native multivectors and materialize
`LateIndex` objects from stored rows.

Simple scalar `where=` filters are pushed into Qdrant before load.

### `kayak.WeaviateLateStore(persistence_path=None, *, client=None, collection_name='LateDocument', vector_name='colbert', environment_variables=None)`

Persist documents in a Weaviate collection with a named self-provided multivector
and materialize `LateIndex` objects from stored rows.

`where=` is applied exactly, but after collection iteration in the current public adapter.

### `kayak.ChromaLateStore(path=None, *, client=None, collection_name='late_documents')`

Persist documents in a Chroma collection and materialize exact `LateIndex`
objects from stored rows.

The adapter stores one pooled dense vector per document in Chroma plus the exact
token matrix in metadata so Kayak can reconstruct the exact index.

### `kayak.register_store(kind, factory, *, replace=False)`

Register a custom store factory behind `open_store(...)`.

## Text Retrievers

### `kayak.LateTextRetriever`

High-level workflow object for text ingest plus search.

Important methods:

- `upsert_texts(doc_ids, texts, metadata=None)`
- `delete(doc_ids)`
- `close()`
- `load_index(...)`
- `search_text(...)`
- `search_query(...)`
- `search_text_batch(...)`
- `search_query_batch(...)`
- `search_text_with_plan(...)`
- `search_query_with_plan(...)`

`load_index(...)` is the reusable exact slice. Use it when the store stays fixed
and the queries are the thing changing.

The public retriever/store contract does not currently promise generic
thread-safe concurrent use of the same store instance across all adapters.
The verified reusable path is:

- call `load_index(...)` once
- reuse that `LateIndex` with `search(...)` or `search_batch(...)`

If you need one explicit same-process multi-caller surface around a fixed
hosted snapshot, use `import kayak_engine` and
`prepare_exact_search_runtime(...)`.

### `LateQuery.to_layout(layout)`

Convert a query between:

- `"nested"`
- `"flat_dim128"`

`"flat_dim128"` requires `vector_dim == 128`.

### `LateIndex.to_layout(layout)`

Convert an index between:

- `"packed"`
- `"hybrid_flat_dim128"`

`"hybrid_flat_dim128"` requires `vector_dim == 128`.

### `LateIndex.select(doc_ids)`

Select a subset of documents and return a new index in the same layout.

This is the mechanism search plans use when they materialize candidate windows
for exact reranking.

## Exact Operations

### `kayak.maxsim(query, index, *, backend=kayak.NUMPY_REFERENCE_BACKEND)`

Return exact scores for every document in the index.

```python
scores = kayak.maxsim(
    query,
    index,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

Returns `LateScores`.

### `kayak.maxsim_batch(batch, index, *, backend=kayak.NUMPY_REFERENCE_BACKEND)`

Return one `LateScores` object per query in the batch.

### `kayak.search(query, index, *, k, backend=kayak.NUMPY_REFERENCE_BACKEND)`

Return exact top-k `SearchHit` tuples.

```python
hits = kayak.search(
    query,
    index,
    k=10,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

### `kayak.search_batch(batch, index, *, k, backend=kayak.NUMPY_REFERENCE_BACKEND)`

Return exact top-k hits for each query in the batch.

## Candidate Generation And Plans

### `kayak.generate_candidates(query, index, generator, *, k, backend=kayak.NUMPY_REFERENCE_BACKEND)`

Run stage-1 candidate generation directly.

```python
result = kayak.generate_candidates(
    query,
    index,
    kayak.document_proxy_candidate_generator(),
    k=100,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

Returns `CandidateStageResult`.

### `kayak.search_with_plan(query, index, plan, *, backend=kayak.NUMPY_REFERENCE_BACKEND)`

Run an explicit search plan and return `SearchPlanResult`.

```python
result = kayak.search_with_plan(
    query,
    index,
    plan,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

## Read Next

- [Usage Patterns](usage-patterns.md) when you want the shortest path from task to API
- [Text Encoders](text-encoders.md) when you need to choose between ColBERT and your own model
- [Storage + Search](storage-and-search.md) when persistence already lives elsewhere
- [Search Plans](search-plans.md) when you need explicit stage reasoning
- [Hosted Engine Python](hosted-engine-python.md) when your real target is `import kayak_engine`

## Plan Builders

### `kayak.exact_full_scan_search_plan(final_k, *, candidate_k=None, stage2_reference_operator=None, stage3_verifier=None)`

Build an exact full-scan plan.

This is the correctness baseline.

### `kayak.exact_full_scan_clause_text_search_plan(final_k, *, candidate_k=None)`

Build an exact full-scan plan with clause-text stage-3 verification.

### `kayak.document_proxy_search_plan(final_k, candidate_k, *, query_vector_budget=0, document_vector_budget=0, stage2_reference_operator=None, stage3_verifier=None)`

Build an approximate-candidate plus exact-rerank plan.

This is the main staged retrieval API.

## Stage Components

### Candidate generators

- `kayak.exact_full_scan_candidate_generator()`
- `kayak.document_proxy_candidate_generator(*, query_vector_budget=0, document_vector_budget=0)`

### Stage-2 reference operators

- `kayak.exact_late_interaction_stage2_reference_operator()`
- `kayak.noop_topk_stage2_reference_operator()`

### Stage-3 verifiers

- `kayak.clause_text_stage3_verifier_operator()`
- `kayak.none_stage3_verifier_operator()`

## Backends

### Constants

```python
kayak.NUMPY_REFERENCE_BACKEND
kayak.MOJO_EXACT_CPU_BACKEND
```

### `kayak.available_backends()`

Return a tuple of available backend names.

```python
('numpy_reference', 'mojo_exact_cpu')
```

or, if Mojo is unavailable:

```python
('numpy_reference',)
```

### `kayak.backend_info(name)`

Return a `BackendInfo` record with:

- `name`
- `available`
- `requires_mojo`
- `query_layouts`
- `index_layouts`
- `availability_reason`

Use this instead of guessing why the Mojo backend is or is not available.

## Types

### Core objects

- `LateQuery`
- `LateQueryBatch`
- `LateDocuments`
- `LateIndex`

### Scoring and hits

- `LateScores`
- `SearchHit`

### Plan and stage results

- `SearchPlan`
- `SearchPlanResult`
- `CandidateStageResult`
- `SearchStageProfile`
- `StageArtifactMaterialization`

### Descriptors

- `CandidateGenerator`
- `Stage2ReferenceOperator`
- `Stage3VerifierOperator`
- `ReferenceScoringSemantics`
- `BackendInfo`

## Notes On Mojo Usage

If you want your own codebase to behave as "use Mojo by default," define a
backend constant and thread it through the operations you call:

```python
BACKEND = kayak.MOJO_EXACT_CPU_BACKEND
```

That is the supported way to make Mojo your application default without hiding
which executor is running.
