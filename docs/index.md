# kayak

**Late-interaction retrieval for Python.**

Kayak makes token-level MaxSim scoring programmable — the retrieval method behind ColBERT. It keeps the structure of late interaction explicit instead of hiding it behind a generic tensor API.

---

## The problem with existing tools

Most vector search APIs work like this:

```python
results = db.search(vector, k=10)  # what happened inside?
```

You get results. You don't see how many document vectors were scored, what the candidate window looked like before reranking, or which stage cost the most.

With late interaction, that opacity is a problem. Token-level MaxSim is more expressive than single-vector search, but query vector count, document vector count, and candidate window size all directly affect latency, memory, and retrieval quality. You need to see them to reason about them.

---

## How kayak is different

Kayak gives you a **search plan** — an explicit, inspectable object that describes every stage of the retrieval pipeline.

```python
plan = kayak.document_proxy_search_plan(
    final_k=10,
    candidate_k=100,
    query_vector_budget=32,
    document_vector_budget=64,
)

result = kayak.search_with_plan(query, index, plan)

result.hits                                        # final results
result.candidate_stage.hits                        # before reranking
result.stage2_reference.query_vector_count         # vectors used
result.stage2_reference.document_vector_count      # doc vectors scored
```

Every stage is visible. Nothing is hidden in a config string or applied server-side without your knowledge.

---

## Install

```bash
pip install kayak
```

Python 3.11+. No GPU required.

→ [Full installation guide](installation.md)

---

## When to use kayak

- You're working with **ColBERT-style embeddings** and need token-level MaxSim
- You want to **understand the trade-offs** between latency, quality, and memory at each retrieval stage
- You need a path from **local experimentation to production** without changing your mental model
- You want retrieval results you can **explain and debug**, not just rankings
