# Vector Databases

The goal is not to replace the database. Keep the database for storage and routing where it helps, then let Kayak own exact late-interaction search on the materialized slice or candidate window.

## Best-Fit Summary

| Store | Best fit right now | Default assumption |
| --- | --- | --- |
| PgVector | Postgres is already your durable store and you want exact filtered slices locally | use `load_index(...)` first, not a custom bridge |
| Qdrant | you want native multivectors and optional candidate handoff | native storage is a strong fit, but still measure whether a full loaded slice is simpler |
| Weaviate | you want named self-provided multivectors or embedded demos | current public adapter is exact, but `where=` is not DB-side pushdown yet |
| LanceDB | you want local file-backed multivector storage and a straightforward exact slice | a one-time load plus Kayak exact search is the default thing to test first |
| Chroma | you already rely on dense retrieval there and want Kayak exact reranking or exact loaded slices | Chroma is a compatibility path, not a native late-interaction engine |
| Milvus | you want the documented handoff pattern more than a finished public walkthrough | treat it as an integration shape, not a shipped notebook |

## Public Store Semantics

All public store adapters support the same base contract: `upsert(...)`, `delete(...)`, `load_index(...)`, `close()`, and context-manager use.

What differs is vector storage shape and how much of `where=` can be applied
before Kayak materializes the exact index.

| Store | Native multivectors | `load_index(where=...)` behavior | Stored exact token matrix |
| --- | --- | --- | --- |
| PgVector | yes | simple scalar filters are pushed into Postgres JSONB before load | yes |
| Qdrant | yes | simple scalar filters are pushed into Qdrant before load | yes |
| Weaviate | yes | current public adapter filters after collection iteration | yes |
| LanceDB | yes | current public adapter filters after Arrow materialization | yes |
| Chroma | no | simple scalar filters are pushed into Chroma before load | yes |

"Simple scalar filters" means the exact-match mapping form shown in the examples, with string, bool, int, or float values.

## Open The Adapter

| Store | Public constructor shape |
| --- | --- |
| PgVector | `kayak.open_store("pgvector", dsn=... \| connection=..., table_name=..., schema_name=...)` |
| Qdrant | `kayak.open_store("qdrant", client=... \| path=..., collection_name=...)` |
| Weaviate | `kayak.open_store("weaviate", client=... \| persistence_path=..., collection_name=..., vector_name=...)` |
| LanceDB | `kayak.open_store("lancedb", path=..., table_name=...)` |
| Chroma | `kayak.open_store("chromadb", client=... \| path=..., collection_name=...)` |

## Notebooks

- [lancedb-to-kayak-reranking.ipynb](notebooks/lancedb-to-kayak-reranking.ipynb)
- [pgvector-to-kayak-exact-search.ipynb](notebooks/pgvector-to-kayak-exact-search.ipynb)
- [qdrant-to-kayak-reranking.ipynb](notebooks/qdrant-to-kayak-reranking.ipynb)
- [weaviate-to-kayak-reranking.ipynb](notebooks/weaviate-to-kayak-reranking.ipynb)
- [chromadb-to-kayak-reranking.ipynb](notebooks/chromadb-to-kayak-reranking.ipynb)

Each notebook prints local timings for stage-1 database retrieval, database retrieval plus Kayak rerank handoff, and full Kayak exact search on the same local slice. Treat those numbers as local usage examples, not decision-grade universal benchmarks.

## Minimal Example: PgVector

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

## Minimal Example: Qdrant Candidate Handoff

```python
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
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

To hand candidates to Kayak, you need query token vectors, candidate document
token vectors, stable document ids, and document text if you want `clause_text`.

## Operational Bias

If you are unsure which pattern to use, test this order first:

1. one filtered exact slice loaded into Kayak
2. exact Kayak search on that slice
3. only then a database candidate stage plus exact rerank

The candidate stage adds another moving part. If the slice already fits locally,
full Kayak exact search can be simpler — current local evidence already includes
cases where the full exact Kayak path is as good or better than the rerank
handoff shape.
