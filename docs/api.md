# API Reference

All public exports come from:

```python
import kayak
```

This page focuses on the stable Python surface and the backend contract.

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

### `kayak.open_store(kind, **kwargs)`

Open a public persistence adapter from the stable factory.

```python
store = kayak.open_store("kayak", path="./kayak-index")
```

Built-in kinds:

- `"memory"`
- `"kayak"` and `"directory"`
- `"lancedb"`

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

### `kayak.ColBERTTextEncoder(model_name='colbert-ir/colbertv2.0', *, checkpoint=None, gpus=0)`

Encode text with a ColBERT checkpoint into `LateQuery` and `LateDocuments`.

### `kayak.register_encoder(kind, factory, *, replace=False)`

Register a custom encoder factory behind `open_encoder(...)`.

## Stores

### `kayak.MemoryLateStore()`

Keep late-interaction documents in memory and materialize `LateIndex` objects on demand.

### `kayak.DirectoryLateStore(path)`

Persist one local packed Kayak snapshot on disk and materialize it back into `LateIndex`.

### `kayak.LanceDBLateStore(path, *, table_name='late_documents')`

Persist documents in LanceDB row storage and materialize `LateIndex` objects from the table.

### `kayak.register_store(kind, factory, *, replace=False)`

Register a custom store factory behind `open_store(...)`.

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
