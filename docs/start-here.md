# Choose Your Path

Most users start from a situation, not from an API name.

## By situation

| Situation | Start here |
| --- | --- |
| I want one working search first | [Installation](installation.md) → [Quickstart](quickstart.md) |
| I start from raw text | [Text Encoders](text-encoders.md) → [Usage Patterns](usage-patterns.md) |
| I already have token vectors | [Quickstart](quickstart.md) → [Python API](api.md) |
| I already have a vector database | [Storage + Search](storage-and-search.md) → [Vector Databases](vector-databases.md) |
| I need repeated-query speed | [Usage Patterns](usage-patterns.md) |

## By API shape

| Situation | First API to look at |
| --- | --- |
| raw text in, search results out | `kayak.open_text_retriever(...)` |
| precomputed vectors in, exact search out | `kayak.query(...)`, `kayak.documents(...).pack()`, `kayak.search(...)` |
| database keeps persistence | `kayak.open_store(...)` plus `store.load_index(...)` |
| repeated queries on one fixed slice | `store.load_index(...)` once, then `kayak.search_batch(...)` |

## Shortest reading order

1. [Installation](installation.md) — verify the exact backend is available before anything else
2. [Quickstart](quickstart.md) — run one working example before tuning adapters, layouts, or plans
3. [Usage Patterns](usage-patterns.md) — pick the right long-term SDK shape

## Common confusions

| If you are thinking... | Open... |
| --- | --- |
| "How do I inspect available backends?" | [Installation](installation.md) |
| "Can I pass a Hugging Face model?" | [Text Encoders](text-encoders.md) |
| "Can I keep Postgres, Qdrant, LanceDB, Weaviate, or Chroma?" | [Storage + Search](storage-and-search.md) |
| "What is the fastest repeated-query path?" | [Usage Patterns](usage-patterns.md) |
| "I need runnable artifacts, not prose." | [Examples](examples.md) |
