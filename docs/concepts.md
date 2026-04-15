# Late Interaction

Understanding what kayak computes — and why it's different from standard dense retrieval.

---

## Dense retrieval vs. late interaction

**Dense retrieval** compresses a query into a single vector, compresses each document into a single vector, and ranks by dot product similarity. Fast, simple, but lossy — all the nuance in both query and document collapses to a single number before comparison.

**Late interaction** keeps all token vectors intact and compares them directly. The query and document meet late — after encoding, not before.

---

## MaxSim

The late interaction scoring function is **MaxSim**:

For each query token vector, find the document token it is most similar to. Sum those maximum similarities across all query tokens.

```
score(query, doc) = Σ  max  sim(q_i, d_j)
                   i∈Q  j∈D
```

In code:

```python
scores = kayak.maxsim(query, index)
```

This means:

- Every query token gets to "find" its best match in the document
- A document scores well if it has good coverage across *most* query tokens
- Token order doesn't matter — it's a maximum over all document positions

---

## Why vector counts matter

MaxSim scales with `query_vectors × document_vectors` per document.

If your query has 32 token vectors and your document has 128 token vectors, scoring one document takes 32 × 128 = 4,096 dot products. Over a million documents, that's 4 billion operations for a full scan.

This is why kayak keeps vector counts explicit everywhere — they are not implementation details, they are retrieval budget.

```python
result.stage2_reference.query_vector_count     # how many query vectors were used
result.stage2_reference.document_vector_count  # how many doc vectors were scored
```

---

## Candidate generation

For large corpora, scoring every document with full MaxSim is too slow. The standard approach: generate a smaller candidate set first, then rerank with exact MaxSim.

Kayak calls this **Stage 1** (candidate generation) and **Stage 2** (exact reranking). Both stages are explicit — you choose them, you see their costs.

→ [Search Plans](search-plans.md)

---

## ColBERT embeddings

Kayak is designed for ColBERT-style token embeddings: 128-dimensional float32 vectors, typically 32–128 vectors per query and 32–256 vectors per document.

The `flat_dim128` and `hybrid_flat_dim128` layouts are optimized for this shape:

```python
flat_query   = query.to_layout("flat_dim128")
hybrid_index = index.to_layout("hybrid_flat_dim128")

scores = kayak.maxsim(flat_query, hybrid_index)
```

Both layouts require `vector_dim == 128`.
