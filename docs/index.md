---
hide:
  - toc
---

# Kayak

Exact late-interaction retrieval for Python. Kayak gives you an explicit surface for exact late-interaction search backed by a real Mojo CPU path, and a practical way to keep your current vector database as the durable system of record.

[Get started](installation.md){ .md-button .md-button--primary }
[Quickstart](quickstart.md){ .md-button }
[Python API](api.md){ .md-button }

## Common workflows

**Local text retrieval** — input starts as raw text, one object owns encoding and search. → [Text Encoders](text-encoders.md), [Usage Patterns](usage-patterns.md)

**Exact search from vectors** — you already own token-level query and document vectors, search directly. → [Quickstart](quickstart.md), [Python API](api.md)

**Database handoff** — LanceDB, PgVector, Qdrant, Weaviate, or Chroma already owns persistence; Kayak owns the search step. → [Storage + Search](storage-and-search.md), [Vector Databases](vector-databases.md)

**Repeated-query serving** — the slice stays fixed, query traffic is the variable. → [Usage Patterns](usage-patterns.md), [Hosted Engine Python](hosted-engine-python.md)

## Where to start

| If you want to... | Open... |
| --- | --- |
| choose by situation instead of API name | [Choose Your Path](start-here.md) |
| install Kayak and verify the exact backend | [Installation](installation.md) |
| run the shortest possible exact search | [Quickstart](quickstart.md) |
| pass a Hugging Face ColBERT checkpoint or your own model | [Text Encoders](text-encoders.md) |
| keep an existing vector database | [Storage + Search](storage-and-search.md) |
| compare adapter-specific behavior | [Vector Databases](vector-databases.md) |
| open runnable notebooks and scripts | [Examples](examples.md) |
| inspect the stable public signatures | [Python API](api.md) |

Start with [Installation](installation.md) → [Quickstart](quickstart.md) → [Usage Patterns](usage-patterns.md).
