# Examples

Real, runnable Kayak examples. Use this page when you want to inspect working code and measured outputs instead of reading conceptual pages first.

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

## Next

| If you want to... | Open... |
| --- | --- |
| get the shortest Mojo-first script | [Quickstart](quickstart.md) |
| choose between retrievers, loaded slices, and plans | [Usage Patterns](usage-patterns.md) |
| understand the database handoff model | [Storage + Search](storage-and-search.md) |
| inspect adapter-specific storage facts | [Vector Databases](vector-databases.md) |
