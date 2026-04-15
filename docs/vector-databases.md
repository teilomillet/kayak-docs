# Vector Database Integrations

This page shows concrete ways to plug Kayak into a vector database.

The main integration shape is:

- let the database own persistence and filtering
- let Kayak own retrieval and surfacing
- use full Kayak search when the searchable slice already fits
- add a database candidate stage only when you actually need it

If you want the explicit “keep the DB for saving, let Kayak search” story, read
[Storage + Search](storage-and-search.md) first.

## Executed Notebooks

These notebooks are included with the docs:

- [lancedb-to-kayak-reranking.ipynb](notebooks/lancedb-to-kayak-reranking.ipynb)
- [pgvector-to-kayak-exact-search.ipynb](notebooks/pgvector-to-kayak-exact-search.ipynb)
- [qdrant-to-kayak-reranking.ipynb](notebooks/qdrant-to-kayak-reranking.ipynb)
- [weaviate-to-kayak-reranking.ipynb](notebooks/weaviate-to-kayak-reranking.ipynb)
- [chromadb-to-kayak-reranking.ipynb](notebooks/chromadb-to-kayak-reranking.ipynb)

Each notebook prints local average timings for:

- stage-1 database retrieval
- database retrieval plus Kayak rerank handoff
- full Kayak exact search on the same local slice

Treat those numbers as local usage examples, not as decision-grade benchmarks.

Script examples in the repository:

- `python/examples/pgvector_store.py`

## Adapter Semantics

All public store adapters support the same basic contract:

- `upsert(...)`
- `delete(...)`
- `load_index(...)`
- `close()`
- `with kayak.open_store(...) as store: ...`

What differs is how each adapter stores vectors and how much of `where=` can be
applied before Kayak materializes the exact index.

| Store | Native multivectors | `load_index(where=...)` behavior | Stored exact token matrix |
| --- | --- | --- | --- |
| PgVector | Yes | simple scalar filters are pushed into Postgres JSONB before load | yes |
| Qdrant | Yes | simple scalar filters are pushed into Qdrant before load | yes |
| Weaviate | Yes | current public adapter filters after collection iteration | yes |
| LanceDB | Yes | current public adapter filters after Arrow materialization | yes |
| Chroma | No | simple scalar filters are pushed into Chroma before load | yes |

For this page, "simple scalar filters" means the exact-match mapping form shown
in the examples, with string, bool, int, or float values.

## Optional Candidate Handoff

This is the common pattern when you want the database to return a candidate
window before Kayak exact search.

```python
import numpy as np
import kayak

BACKEND = kayak.MOJO_EXACT_CPU_BACKEND

# stage 1: the database returns candidates
candidate_rows = vector_db.search(query_vectors).limit(50)

candidate_index = kayak.documents(
    [row["doc_id"] for row in candidate_rows],
    [np.asarray(row["vector"], dtype=np.float32) for row in candidate_rows],
    texts=[row["text"] for row in candidate_rows],
).pack()

query = kayak.query(query_vectors, text=query_text)

hits = kayak.search(
    query,
    candidate_index,
    k=10,
    backend=BACKEND,
)
```

To hand candidates to Kayak, you need:

- query token vectors
- candidate document token vectors
- stable document ids
- document text if you want `clause_text`

## PgVector / Postgres

PgVector is the direct Postgres path when you want to keep rows in your
existing database and let Kayak materialize the exact slice locally.

Install the adapter dependencies:

```bash
uv add "psycopg[binary]" pgvector
```

or:

```bash
pixi add --pypi "psycopg[binary]" pgvector
```

Then use the public store adapter:

```python
store = kayak.open_store(
    "pgvector",
    dsn="postgresql://postgres:postgres@127.0.0.1:5432/postgres",
    table_name="docs",
)
store.upsert(documents, metadata=metadata_rows)

index = store.load_index(where={"tenant": "acme"}, include_text=True)
hits = kayak.search(
    query,
    index,
    k=10,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

The current public adapter stores:

- one exact token matrix per document as `vector(dim)[]`
- optional document text in `text`
- exact metadata in `jsonb`

That means you can keep Postgres as the durable store, apply simple scalar
metadata filters before load, and still let Kayak own the exact late-interaction
search step.

## Qdrant

Qdrant is a direct fit for ColBERT-style multivectors.

Kayak now exposes a direct public store adapter for the "keep Qdrant for
storage, let Kayak search" path:

```python
store = kayak.open_store(
    "qdrant",
    client=my_qdrant_client,
    collection_name="docs",
)
store.upsert(documents, metadata=metadata_rows)

index = store.load_index(where={"tenant": "acme"}, include_text=True)
hits = kayak.search(
    query,
    index,
    k=10,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

The executed notebook is:

- [qdrant-to-kayak-reranking.ipynb](notebooks/qdrant-to-kayak-reranking.ipynb)

It shows:

1. a local `:memory:` Qdrant collection
2. multivector storage with `MAX_SIM`
3. metadata filtering in Qdrant
4. fetching candidate vectors with `with_vectors=True`
5. building a Kayak candidate index from those results
6. exact reranking and clause-text verification in Kayak
7. local timing for Qdrant stage 1, Qdrant plus Kayak, and full Kayak exact search

Minimal shape:

```python
from qdrant_client import QdrantClient, models
import numpy as np
import kayak

client = QdrantClient(":memory:")

client.create_collection(
    collection_name="docs",
    vectors_config=models.VectorParams(
        size=128,
        distance=models.Distance.COSINE,
        multivector_config=models.MultiVectorConfig(
            comparator=models.MultiVectorComparator.MAX_SIM,
        ),
    ),
)

results = client.query_points(
    collection_name="docs",
    query=query_vectors.tolist(),
    query_filter=models.Filter(
        must=[
            models.FieldCondition(
                key="category",
                match=models.MatchValue(value="installation"),
            )
        ]
    ),
    with_vectors=True,
    limit=20,
).points

candidate_index = kayak.documents(
    [point.payload["doc_id"] for point in results],
    [np.asarray(point.vector, dtype=np.float32) for point in results],
    texts=[point.payload["text"] for point in results],
).pack()
```

Use Qdrant this way when you want:

- multivector persistence
- database-side filtering before reranking
- a clean handoff from native multivector retrieval into Kayak exact local evaluation

## Weaviate

Weaviate also supports self-provided multivectors.

Kayak now exposes a direct public store adapter for the same storage-first
shape:

```python
store = kayak.open_store(
    "weaviate",
    client=my_weaviate_client,
    collection_name="Doc",
    vector_name="colbert",
)
store.upsert(documents, metadata=metadata_rows)

index = store.load_index(where={"tenant": "acme"}, include_text=True)
hits = kayak.search(
    query,
    index,
    k=10,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

The executed notebook is:

- [weaviate-to-kayak-reranking.ipynb](notebooks/weaviate-to-kayak-reranking.ipynb)

It shows:

1. a local embedded Weaviate instance
2. a collection with `Configure.MultiVectors.self_provided(name="colbert")`
3. inserting token matrices as the named multivector
4. filtered `near_vector` search
5. returning candidate vectors with `include_vector=["colbert"]`
6. exact Kayak reranking and clause-text verification
7. local timing for Weaviate stage 1, Weaviate plus Kayak, and full Kayak exact search

Minimal shape:

```python
import weaviate
import weaviate.classes as wvc
import numpy as np
import kayak

client = weaviate.connect_to_embedded(
    persistence_data_path="/tmp/weaviate-kayak",
)

collection = client.collections.create(
    name="Doc",
    properties=[
        wvc.config.Property(name="doc_id", data_type=wvc.config.DataType.TEXT),
        wvc.config.Property(name="category", data_type=wvc.config.DataType.TEXT),
        wvc.config.Property(name="text", data_type=wvc.config.DataType.TEXT),
    ],
    vector_config=wvc.config.Configure.MultiVectors.self_provided(name="colbert"),
)

result = collection.query.near_vector(
    near_vector={"colbert": query_vectors.tolist()},
    target_vector="colbert",
    filters=wvc.query.Filter.by_property("category").equal("installation"),
    limit=20,
    include_vector=["colbert"],
    return_properties=["doc_id", "category", "text"],
)

candidate_index = kayak.documents(
    [obj.properties["doc_id"] for obj in result.objects],
    [np.asarray(obj.vector["colbert"], dtype=np.float32) for obj in result.objects],
    texts=[obj.properties["text"] for obj in result.objects],
).pack()
```

Use Weaviate this way when you want:

- embedded local demos
- named multivectors with self-provided embeddings
- a straightforward candidate-vector handoff into Kayak

For the current public adapter, `load_index(where=...)` is exact but it is not a
database-side pushdown path yet. The adapter iterates the collection and applies
the metadata filter before materializing the Kayak index.

## LanceDB

LanceDB can store multivectors directly and works well as a local stage-1 store.

The executed notebook is:

- [lancedb-to-kayak-reranking.ipynb](notebooks/lancedb-to-kayak-reranking.ipynb)

Use LanceDB when you want:

- a local file-backed table
- Arrow-native tooling around the candidate set
- a simple multivector storage path for notebooks and demos

Kayak also exposes a direct public store adapter for the "keep LanceDB for
storage, let Kayak search" path:

```python
import kayak

store = kayak.open_store(
    "lancedb",
    path="./lancedb-store",
    table_name="docs",
)
store.upsert(documents, metadata=metadata_rows)

index = store.load_index(where={"tenant": "acme"}, include_text=True)
hits = kayak.search(
    query,
    index,
    k=10,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

That path is useful when:

- LanceDB remains the durable row store
- Kayak owns exact late-interaction scoring
- you want one factual store contract instead of a custom bridge per application

For the current public adapter, `load_index(where=...)` is exact but it filters
after Arrow materialization, not inside LanceDB itself.

## ChromaDB

Chroma is a practical dense stage-1 store, not a native late-interaction store.

Kayak now exposes a compatibility store adapter for the same public contract:

```python
store = kayak.open_store(
    "chromadb",
    client=my_chroma_client,
    collection_name="docs",
)
store.upsert(documents, metadata=metadata_rows)

index = store.load_index(where={"tenant": "acme"}, include_text=True)
hits = kayak.search(
    query,
    index,
    k=10,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

The executed notebook is:

- [chromadb-to-kayak-reranking.ipynb](notebooks/chromadb-to-kayak-reranking.ipynb)

That notebook uses:

1. one pooled dense vector per document inside Chroma
2. a sidecar Python store for the token-level document vectors
3. dense Chroma retrieval for candidates
4. optional exact Kayak reranking on those candidates
5. a timing comparison between:
   dense-only Chroma
   Chroma plus Kayak rerank
   full Kayak exact search

The measured result in that notebook is intentionally factual:

- on the local example slice, full Kayak exact search was the fastest path
- the Chroma rerank path was slower
- the rerank step did not change the top results on that slice

Use the Chroma rerank pattern when you actually need:

- Chroma as the storage layer
- Chroma metadata filtering
- Chroma candidate generation on a larger collection

If your filtered slice is already small, the notebook shows why a full exact
Kayak search can be the simpler and faster path.

The public Chroma store adapter stores:

- one pooled dense vector per document for Chroma's native dense collection
- the exact token matrix serialized in metadata so Kayak can reconstruct the exact index

That is why the adapter can satisfy the same `load_index(...)` contract without
pretending Chroma is a native late-interaction engine.

Minimal shape:

```python
import chromadb
import numpy as np
import kayak

client = chromadb.Client()
collection = client.create_collection(name="docs")

# stage 1 stores one dense vector per document
collection.add(
    ids=doc_ids,
    embeddings=[doc_dense_vector.tolist() for doc_dense_vector in dense_vectors],
    documents=doc_texts,
    metadatas=doc_metadatas,
)

# token-level vectors stay in a sidecar store for Kayak
token_store = {
    doc_id: token_vectors
    for doc_id, token_vectors in zip(doc_ids, document_token_vectors)
}

result = collection.query(
    query_embeddings=[dense_query_vector.tolist()],
    where={"topic": "installation"},
    n_results=20,
    include=["metadatas", "documents"],
)

candidate_ids = [metadata["doc_id"] for metadata in result["metadatas"][0]]
candidate_index = kayak.documents(
    candidate_ids,
    [token_store[doc_id] for doc_id in candidate_ids],
    texts=[text_store[doc_id] for doc_id in candidate_ids],
).pack()

hits = kayak.search(query, candidate_index, k=10, backend=kayak.MOJO_EXACT_CPU_BACKEND)
```

## Milvus

Milvus exposes multi-vector query primitives in the Python client, including
`EmbeddingList`.

The useful Milvus pattern is the same:

1. store token-level vectors in the Milvus collection
2. query the vector field with an `EmbeddingList`
3. fetch candidate ids, vectors, and text
4. build a Kayak candidate index
5. run exact Kayak search or `search_with_plan`

An executed Milvus notebook is not included yet, so treat the Milvus section as
a documented server-side integration pattern.

## Metadata Filtering In The Database

Keep filtering in the database when that is already the system of record.

```python
candidate_rows = (
    vector_db.search(query_vectors)
    .where("category = 'installation'")
    .limit(20)
)

candidate_index = kayak.documents(
    [row["doc_id"] for row in candidate_rows],
    [np.asarray(row["vector"], dtype=np.float32) for row in candidate_rows],
    texts=[row["text"] for row in candidate_rows],
).pack()

hits = kayak.search(
    query,
    candidate_index,
    k=5,
    backend=BACKEND,
)
```

This keeps:

- filtering
- tenancy
- collection routing
- database-specific access control

in the database, while Kayak handles exact late-interaction scoring on the
remaining slice.

## Exact Baseline Beside The Database

Use a full Kayak index when you want to measure how much quality the database
candidate stage is preserving.

```python
full_index = kayak.documents(
    all_doc_ids,
    all_document_vectors,
    texts=all_document_texts,
).pack()

exact_hits = kayak.search(
    query,
    full_index,
    k=10,
    backend=BACKEND,
)
```

Use this for:

- offline quality checks
- regression tests
- comparing candidate-window settings
- validating whether the database stage is surfacing the exact top-k

## Stage Profiles On Top Of Database Candidates

When you want auditable stage counts after the database has already chosen the
candidate set, run a plan over the candidate index.

```python
plan = kayak.exact_full_scan_search_plan(
    final_k=5,
    candidate_k=len(candidate_rows),
    stage3_verifier=kayak.clause_text_stage3_verifier_operator(),
)

result = kayak.search_with_plan(
    query,
    candidate_index,
    plan,
    backend=BACKEND,
)

print(result.stage2)
print(result.stage3_verifier)
```

This gives you:

- exact top-k hits
- explicit stage accounting
- verifier materialization counts
- document text clauses for the final hits

## What You Need From The Database

Kayak reranking is easiest when the database can return:

- token-level document vectors
- stable ids
- document text
- metadata filters before reranking

If your current database stores only one dense vector per document, Kayak can
still rerank, but you need a second store for the late-interaction vectors.
