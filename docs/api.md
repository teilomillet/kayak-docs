# API Reference

All public exports from `import kayak`.

---

## Creating objects

### `kayak.query(vectors, *, text=None)`

Create a query from a 2D NumPy array `(query_tokens, dim)`.

```python
query = kayak.query(np.array([[1.0, 0.0], [0.0, 1.0]], dtype=np.float32))

# with optional text for stage-3 operators
query = kayak.query(vectors, text="founded in 1984 in a church")
```

### `kayak.documents(doc_ids, vectors_list, *, texts=None)`

Create a document collection.

```python
docs = kayak.documents(
    ["doc-a", "doc-b"],
    [vectors_a, vectors_b],
)

# with texts for clause_text stage-3
docs = kayak.documents(
    ["doc-a", "doc-b"],
    [vectors_a, vectors_b],
    texts=["text of doc a", "text of doc b"],
)
```

### `kayak.query_batch(vectors_list)`

Create a batch of queries. Each query can have a different number of token vectors.

```python
batch = kayak.query_batch([vectors_q1, vectors_q2, vectors_q3])
```

---

## Indexing

### `docs.pack()`

Pack documents into a searchable index.

```python
index = docs.pack()
```

### `index.to_layout(layout)`

Convert to an optimized layout. Requires `vector_dim == 128`.

| Layout | Description |
|--------|-------------|
| `"flat_dim128"` | Flat query layout for 128-dim embeddings |
| `"hybrid_flat_dim128"` | Flat document index layout for 128-dim embeddings |

```python
flat_query   = query.to_layout("flat_dim128")
hybrid_index = index.to_layout("hybrid_flat_dim128")
```

### `kayak.packed_index(doc_ids, vectors_list)`

Shorthand for `kayak.documents(...).pack()`.

### `kayak.hybrid_flat_dim128_index(doc_ids, vectors_list)`

Shorthand for `kayak.documents(...).pack().to_layout("hybrid_flat_dim128")`.

### `kayak.flat_query_dim128(vectors)`

Shorthand for `kayak.query(vectors).to_layout("flat_dim128")`.

---

## Scoring

### `kayak.maxsim(query, index, *, backend=NUMPY_REFERENCE_BACKEND)`

Exact MaxSim scores for all documents in the index.

```python
scores = kayak.maxsim(query, index)

scores.doc_ids   # ('doc-a', 'doc-b', ...)
scores.values    # np.array([2.0, 1.75, ...], dtype=float32)
```

### `kayak.maxsim_batch(batch, index, *, backend=NUMPY_REFERENCE_BACKEND)`

MaxSim for a batch of queries. Returns one `LateScores` per query.

---

## Search

### `kayak.search(query, index, k, *, backend=NUMPY_REFERENCE_BACKEND)`

Top-k search. Returns a tuple of `SearchHit`.

```python
hits = kayak.search(query, index, k=10)
# (SearchHit(doc_id='doc-a', score=2.0), ...)
```

### `kayak.search_with_plan(query, index, plan, *, backend=NUMPY_REFERENCE_BACKEND)`

Stage-aware search. Returns a `SearchPlanResult` with full intermediate state.

```python
result = kayak.search_with_plan(query, index, plan)
```

### `kayak.search_batch(batch, index, k, *, backend=NUMPY_REFERENCE_BACKEND)`

Top-k search for a batch of queries.

---

## Search plans

### `kayak.exact_full_scan_search_plan(final_k, *, candidate_k=None, stage3_verifier=None)`

Exact scan over all documents. Correct by definition, slow on large corpora.

### `kayak.document_proxy_search_plan(final_k, candidate_k, *, query_vector_budget=0, document_vector_budget=0, stage2_reference_operator=None, stage3_verifier=None)`

Approximate candidate generation with exact MaxSim reranking.

### `kayak.exact_full_scan_clause_text_search_plan(final_k, *, candidate_k=None)`

Exact scan with clause-text stage-3 verifier.

---

## Stage operators

### Stage 1 — candidate generators

| Function | Description |
|----------|-------------|
| `kayak.exact_full_scan_candidate_generator()` | Exact scan over all documents |
| `kayak.document_proxy_candidate_generator(*, query_vector_budget=0, document_vector_budget=0)` | Approximate proxy-based candidate generation |

### Stage 2 — reference operators

| Function | Description |
|----------|-------------|
| `kayak.exact_late_interaction_stage2_reference_operator()` | Exact MaxSim reranking |
| `kayak.noop_topk_stage2_reference_operator()` | Pass through top-k from Stage 1 |

### Stage 3 — verifiers

| Function | Description |
|----------|-------------|
| `kayak.clause_text_stage3_verifier_operator()` | Text-aware refinement. Requires `query.text` and document texts. |
| `kayak.none_stage3_verifier_operator()` | No stage-3 verification |

---

## Candidate generation

### `kayak.generate_candidates(query, index, generator, k, *, backend=NUMPY_REFERENCE_BACKEND)`

Run stage-1 candidate generation directly.

```python
result = kayak.generate_candidates(
    query,
    index,
    kayak.document_proxy_candidate_generator(),
    k=100,
)

result.hits             # candidate hits with approximate scores
result.profile          # SearchStageProfile with vector counts
result.candidate_doc_ids  # doc IDs of candidates
```

---

## Backends

```python
kayak.NUMPY_REFERENCE_BACKEND   # 'numpy_reference'
kayak.MOJO_EXACT_CPU_BACKEND    # 'mojo_exact_cpu'

kayak.available_backends()      # list of available backend names
kayak.backend_info(name)        # BackendInfo for a specific backend
```

---

## Types

| Type | Description |
|------|-------------|
| `LateQuery` | A query with explicit token vectors and optional text |
| `LateQueryBatch` | A batch of queries with different token counts |
| `LateDocuments` | A document collection before packing |
| `LateIndex` | A packed, searchable index |
| `LateScores` | Per-document MaxSim scores with `doc_ids` and `values` |
| `SearchHit` | A single result: `doc_id` and `score` |
| `SearchPlan` | An explicit retrieval pipeline description |
| `SearchPlanResult` | Full result with per-stage intermediate state |
| `SearchStageProfile` | Vector and document counts for one stage |
| `CandidateStageResult` | Stage-1 output: hits, scores, and profile |
| `CandidateGenerator` | Stage-1 generator descriptor |
| `Stage2ReferenceOperator` | Stage-2 operator descriptor |
| `Stage3VerifierOperator` | Stage-3 verifier descriptor |
| `StageArtifactMaterialization` | What was materialized in a stage |
| `ReferenceScoringSemantics` | Scoring semantics for stage-2 |
| `BackendInfo` | Backend availability and metadata |
