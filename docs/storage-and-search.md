# Storage + Search

If you already use a vector database, keep it for storage.

The default Kayak shape is simple:

1. keep the database for persistence
2. keep filtering and metadata there if you need them
3. let Kayak own retrieval and surfacing

## Default Recommendation

Use full Kayak retrieval when the searchable slice:

- fits on the target host
- can be refreshed on your normal update cadence
- does not need a database-side candidate step before search

That is usually the simplest production shape:

```python
import numpy as np
import kayak

BACKEND = kayak.MOJO_EXACT_CPU_BACKEND

rows = vector_db.fetch_all()

index = kayak.documents(
    [row["doc_id"] for row in rows],
    [np.asarray(row["vector"], dtype=np.float32) for row in rows],
    texts=[row["text"] for row in rows],
).pack()

query = kayak.query(query_vectors, text=query_text)
hits = kayak.search(query, index, k=10, backend=BACKEND)
```

In this shape:

- the database is the durable store
- Kayak is the search engine

## Public Store Adapter For LanceDB

If the storage layer is LanceDB, you can use the public store adapter directly
instead of writing a row-to-index bridge yourself:

```python
import kayak

encoder = kayak.open_encoder(
    "callable",
    query_encoder=my_query_encoder,
    document_encoder=my_document_encoder,
)

documents = encoder.encode_documents(doc_ids, texts)

store = kayak.open_store(
    "lancedb",
    path="./lancedb-store",
    table_name="docs",
)
store.upsert(documents, metadata=metadata_rows)

index = store.load_index(where={"tenant": "acme"}, include_text=True)
query = encoder.encode_query("install python and mojo together")
hits = kayak.search(
    query,
    index,
    k=10,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

In that shape:

- LanceDB keeps persistence
- Kayak materializes the searchable slice
- Kayak runs exact late-interaction search on the loaded index

## When To Add A Database Candidate Stage

Add a database-side candidate step only when you need it.

Typical reasons:

- the full searchable slice is too large to keep in one search-ready Kayak index
- you need strict tenant routing or filtering before search
- you want the database to reduce the candidate set before exact Kayak scoring

The handoff shape is:

```python
import numpy as np
import kayak

BACKEND = kayak.MOJO_EXACT_CPU_BACKEND

candidate_rows = vector_db.search(query_vectors).limit(50)

candidate_index = kayak.documents(
    [row["doc_id"] for row in candidate_rows],
    [np.asarray(row["vector"], dtype=np.float32) for row in candidate_rows],
    texts=[row["text"] for row in candidate_rows],
).pack()

query = kayak.query(query_vectors, text=query_text)
hits = kayak.search(query, candidate_index, k=10, backend=BACKEND)
```

That keeps the storage layer and the search layer separate:

- the database stores
- Kayak searches

## Measured LanceDB Example

Kayak was measured on encoded BrowseComp-Plus slices using:

- the same stored document vectors
- the same judged queries
- LanceDB as the storage source
- Kayak exact search after a one-time load from LanceDB

Results on the gold slice:

| System | Mean NDCG@10 | Mean search seconds |
| --- | ---: | ---: |
| LanceDB search | `0.28512677790387286` | `0.0256748021444461` |
| Kayak exact search | `0.28512677790387286` | `0.001213687510850529` |

One-time load from LanceDB into Kayak on that slice:

- `0.20880537503398955` seconds

Results on the evidence slice:

| System | Mean NDCG@10 | Mean search seconds |
| --- | ---: | ---: |
| LanceDB search | `0.26234761964459435` | `0.026544423764183495` |
| Kayak exact search | `0.26234761964459435` | `0.0021198958177895597` |

One-time load from LanceDB into Kayak on that slice:

- `0.21453558304347098` seconds

Observed search-time ratios on those measured slices:

- gold slice: Kayak exact search was `21.154376159357273x` faster
- evidence slice: Kayak exact search was `12.52156994765039x` faster

These are measured examples on those slices. They are useful deployment
evidence, not a universal benchmark claim.

## Scaling Example

On additional measured LanceDB comparisons over the same encoded task family,
Kayak exact search kept the same judged quality while search-time ratios stayed
in this range:

- gold slices from `90` to `720` documents: `5.600745563055038x` to `15.380583580921682x`
- evidence slices from `90` to `720` documents: `7.218913860302956x` to `17.651162894217933x`

The practical reading is straightforward:

- if you can load the searchable slice into Kayak, full Kayak search is a strong default
- if you cannot, keep the database as the candidate stage and hand the slice to Kayak

## Choosing The Shape

Use full Kayak exact search when:

- you want the simplest deployment path
- you can refresh the in-process or local Kayak index on your update schedule
- you want Kayak to own retrieval directly

Use a database candidate stage when:

- database-native filtering must happen before search
- the searchable corpus needs a narrower window before exact scoring
- you already operate a database retrieval layer that you want to keep in front

## Worked Examples

For concrete code against different systems, see:

- [Vector Database Integrations](vector-databases.md)
- [API Reference](api.md)
- [lancedb-to-kayak-reranking.ipynb](notebooks/lancedb-to-kayak-reranking.ipynb)
- [qdrant-to-kayak-reranking.ipynb](notebooks/qdrant-to-kayak-reranking.ipynb)
- [weaviate-to-kayak-reranking.ipynb](notebooks/weaviate-to-kayak-reranking.ipynb)
- [chromadb-to-kayak-reranking.ipynb](notebooks/chromadb-to-kayak-reranking.ipynb)
