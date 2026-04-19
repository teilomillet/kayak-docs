# Python API

All stable SDK imports come from:

```python
import kayak
```

This page covers the public Python SDK only.

## Start here

| If you want to... | Use... |
| --- | --- |
| ingest text and search with one object | `kayak.open_text_retriever(...)` |
| choose an encoder explicitly | `kayak.open_encoder(...)` |
| keep your current database for persistence | `kayak.open_store(...)` |
| search from precomputed token vectors | `kayak.query(...)`, `kayak.documents(...).pack()`, `kayak.search(...)` |
| run many queries against one loaded index | `kayak.query_batch(...)`, `kayak.search_batch(...)` |
| inspect candidate stages and reranking | `kayak.document_proxy_search_plan(...)`, `kayak.search_with_plan(...)` |
| inspect backend availability | `kayak.available_backends()`, `kayak.backend_info(...)` |
| inspect install and runtime diagnostics | `kayak.doctor()` |

## Recommendation First

Recommended public usage:

- simplest text path: `kayak.open_text_retriever(...)`
- simplest vector path: `kayak.query(...)` + `kayak.documents(...).pack()`
- fastest repeated-query public path: reuse one `LateIndex` and call `kayak.search_batch(...)`
- lowest-level exact top-k fast path today: `kayak.search(...)` or `kayak.search_batch(...)` with `backend=kayak.MOJO_EXACT_CPU_BACKEND` when that backend is available

Current fast-path conditions in the SDK:

- packed index layout
- nested query layout
- explicit Mojo exact CPU backend selection

Those are execution details. They are not a reason to expose backend-specific application primitives to ordinary SDK users.

## Factories And Discovery

### `kayak.open_text_retriever(...)`

High-level text workflow. It composes one encoder, one store, and one default backend.

### `kayak.open_encoder(kind, **kwargs)`

Open a public encoder factory.

Built-in kinds:

- `"callable"`
- `"colbert"`

Discovery and extension helpers:

- `kayak.available_encoder_kinds()`
- `kayak.register_encoder(kind, factory, *, replace=False)`
- `kayak.DEFAULT_COLBERT_MODEL_NAME`

### `kayak.open_store(kind, **kwargs)`

Open a public persistence adapter.

Built-in kinds:

- `"memory"`
- `"kayak"` and `"directory"`
- `"lancedb"` and `"lance"`
- `"pgvector"`, `"postgres"`, and `"postgresql"`
- `"qdrant"`
- `"weaviate"`
- `"chromadb"` and `"chroma"`

Discovery and extension helpers:

- `kayak.available_store_kinds()`
- `kayak.register_store(kind, factory, *, replace=False)`

## Core Data Objects

| Export | Purpose |
| --- | --- |
| `LateQuery` | one token-level query matrix |
| `LateQueryBatch` | many queries with explicit ragged vector counts |
| `LateDocuments` | document ids plus token matrices, optionally texts |
| `LateIndex` | searchable packed or layout-converted index |
| `LateScores` | full score vector output from `maxsim(...)` |
| `SearchHit` | one ranked hit |
| `LateTextRetriever` | high-level text ingest and search workflow |
| `LateTextSearchSession` | one reusable loaded slice plus repeated query entrypoints |
| `LateStore` | store protocol |
| `LateStoreStats` | measurable store state |
| `StoreCapabilities` | store capability flags |
| `LateTextEncoder` | encoder protocol |

Common constructors:

- `kayak.query(token_vectors, *, text=None)`
- `kayak.query_batch(token_vectors)`
- `kayak.documents(doc_ids, token_vectors, *, texts=None)`
- `LateDocuments.pack()`
- `LateQuery.to_layout(...)`
- `LateIndex.to_layout(...)`

Low-level layout constructors:

- `kayak.packed_index(...)`
- `kayak.flat_query_dim128(...)`
- `kayak.hybrid_flat_dim128_index(...)`

## Search And Scoring

| Export | Purpose |
| --- | --- |
| `kayak.maxsim(...)` | full exact score vector for one query |
| `kayak.maxsim_batch(...)` | full exact score vectors for many queries |
| `kayak.search(...)` | top-k exact hits for one query |
| `kayak.search_batch(...)` | top-k exact hits for many queries |
| `kayak.generate_candidates(...)` | run only the candidate stage from a plan |
| `kayak.search_with_plan(...)` | execute explicit multi-stage search |

Default rule:

- use `search(...)` or `search_batch(...)` when you want ranked hits
- use `maxsim(...)` or `maxsim_batch(...)` only when you need the full score vector

## Plans, Candidate Stages, And Verifiers

Main plan builders:

- `kayak.exact_full_scan_search_plan(...)`
- `kayak.exact_full_scan_clause_text_search_plan(...)`
- `kayak.document_proxy_search_plan(...)`

Main stage components:

- `kayak.exact_full_scan_candidate_generator()`
- `kayak.document_proxy_candidate_generator()`
- `kayak.noop_topk_stage2_reference_operator()`
- `kayak.exact_late_interaction_stage2_reference_operator()`
- `kayak.none_stage3_verifier_operator()`
- `kayak.clause_text_stage3_verifier_operator()`

Main plan result objects:

- `CandidateGenerator`
- `CandidateStageResult`
- `SearchPlan`
- `SearchPlanResult`
- `SearchStageProfile`
- `Stage2ReferenceOperator`
- `Stage3VerifierOperator`
- `StageArtifactMaterialization`
- `ReferenceScoringSemantics`
- `kayak.exact_late_interaction_reference_scoring_semantics()`

Use [Search Plans](search-plans.md) for examples and tradeoffs.

## Encoders

Built-in public encoder classes:

- `kayak.CallableLateTextEncoder(query_encoder, document_encoder)`
- `kayak.ColBERTTextEncoder(model_name='colbert-ir/colbertv2.0', *, checkpoint=None, gpus=0)`

Use [Text Encoders](text-encoders.md) when you need the constructor details.

## Stores

Built-in public store classes:

- `kayak.MemoryLateStore()`
- `kayak.DirectoryLateStore(path)`
- `kayak.LanceDBLateStore(path, *, table_name='late_documents')`
- `kayak.PgVectorLateStore(dsn=None, *, connection=None, table_name='late_documents', schema_name='public')`
- `kayak.QdrantLateStore(path=None, *, client=None, collection_name='late_documents')`
- `kayak.WeaviateLateStore(persistence_path=None, *, client=None, collection_name='LateDocument', vector_name='colbert', environment_variables=None)`
- `kayak.ChromaLateStore(path=None, *, client=None, collection_name='late_documents')`

Use [Storage + Search](storage-and-search.md) and [Vector Databases](vector-databases.md) for adapter-specific behavior.

## Backends And Diagnostics

Backend constants:

- `kayak.NUMPY_REFERENCE_BACKEND`
- `kayak.MOJO_EXACT_CPU_BACKEND`

Backend inspection:

- `kayak.available_backends()`
- `kayak.backend_info(name)`
- `BackendInfo`

Install and runtime diagnostics:

- `kayak.doctor(*, probe_mojo_load=False)`
- `KayakDoctorReport`
- `KayakFeatureStatus`

REPL help:

- `kayak.help(topic=None)`

## Typing

Use `kayak.typing` for stable public type aliases instead of importing from the internal bridge package.

Example:

```python
from kayak.typing import DocIdsInput, DocTextsInput, TokenMatrixInput
```

## Minimal Examples

### Text Workflow

```python
import kayak

retriever = kayak.open_text_retriever(
    encoder="colbert",
    store="memory",
)

retriever.upsert_texts(["doc-1"], ["late interaction retrieval"])
hits = retriever.search_text("late interaction", k=1)
```

### Vector Workflow

```python
import kayak

query = kayak.query(query_vectors)
index = kayak.documents(doc_ids, document_vectors).pack()
hits = kayak.search(query, index, k=5)
```

### One Loaded Slice Reused Many Times

```python
index = store.load_index(where={"tenant": "acme"})
session = kayak.LateTextSearchSession(encoder=encoder, index=index)
hits = session.search_text("query text", k=5)
```
