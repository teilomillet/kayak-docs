# Late Interaction

This page is about the model Kayak exposes, not just the method names.

The SDK is designed so you can answer three questions directly from code:

1. What exactly got scored?
2. How many vectors did that require?
3. Which backend executed the exact path?

## Dense Retrieval Vs. Late Interaction

Dense retrieval compresses the query to one vector and compresses each document
to one vector. Ranking is then one dot product per document.

Late interaction keeps token vectors explicit.

The query and document are compared after encoding, at scoring time, not
collapsed to one embedding first.

That is more expressive, but it makes vector counts a first-class cost axis.

## Late Interaction Is Not Just "Multi-Vector"

One useful distinction from the talk is this:

- many vectors are not enough on their own
- what matters is interaction at query time

If you represent a document with many vectors but still collapse scoring to a
plain pooled similarity, you have not recovered the main property that makes
late interaction interesting.

What matters is that query-time scoring can align query pieces with document
pieces locally, instead of forcing everything through one global document
embedding.

For Kayak's current exact scorer, that local interaction is MaxSim.

## Late Interaction Is Not Just "ColBERT"

ColBERT is an important concrete design point, but the broader idea is more
general than one model family.

The reusable concept is:

1. keep fine-grained query and document representations
2. let them interact at query time
3. make the retrieval pipeline explicit enough to keep that interaction
   searchable at scale

Kayak's Python SDK is built around that broader idea, not around one particular
paper checkpoint or training recipe.

## MaxSim

Kayak's exact scorer is MaxSim:

```text
score(query, doc) = sum over query tokens of the best matching document token
```

More explicitly:

```text
score(query, doc) = sum_i max_j sim(q_i, d_j)
```

In code:

```python
scores = kayak.maxsim(query, index, backend=kayak.MOJO_EXACT_CPU_BACKEND)
```

The important property is that every query token gets its own best match.
That is why token-level embeddings matter here.

Another way to say this is that MaxSim gives you a lightweight alignment
mechanism at query time. It is not full cross-attention, but it is much closer
to local matching than a single document embedding can be.

## Why Vector Counts Matter

Exact late interaction cost scales with:

```text
query_vector_count * document_vector_count
```

per document.

If a query has 32 vectors and a document has 128 vectors, exact scoring that
one document requires 4096 pairwise similarities before the per-query-token
max reductions are summed.

That is why Kayak keeps vector counts explicit in the API and the plan results.

```python
result.stage2_reference.query_vector_count
result.stage2_reference.document_vector_count
```

Those fields are not metadata decoration. They are part of the retrieval budget.

## Layout Is Explicit Too

Kayak separates the semantic object model from the storage layout:

- `LateQuery` can be `nested` or `flat_dim128`
- `LateIndex` can be `packed` or `hybrid_flat_dim128`

For 128-dimensional embeddings:

```python
flat_query = query.to_layout("flat_dim128")
hybrid_index = index.to_layout("hybrid_flat_dim128")
```

Both conversions require `vector_dim == 128`.

That layout choice matters for profiling, backend execution, and reproducible
benchmark setup, so the SDK keeps it visible.

## Backend Choice Changes The Executor, Not The Math

Kayak exposes two backends:

- `kayak.NUMPY_REFERENCE_BACKEND`
- `kayak.MOJO_EXACT_CPU_BACKEND`

The backend should not change the scoring semantics. It changes which executor
runs the exact path.

That distinction matters for documentation and benchmarking:

- compare scores across backends when validating correctness
- compare latency only after the semantic path is fixed
- do not describe Mojo as a different retrieval algorithm

## Candidate Generation And Reranking

Exact MaxSim over every document is the correctness baseline.
It is also expensive.

That leads to the usual staged retrieval pattern:

1. Stage 1 narrows the candidate set.
2. Stage 2 runs exact late interaction on those candidates.
3. Stage 3 can optionally do text-aware verification.

In Kayak those stages are explicit objects and explicit profiles:

```python
plan = kayak.document_proxy_search_plan(
    final_k=10,
    candidate_k=100,
    query_vector_budget=32,
    document_vector_budget=64,
)

result = kayak.search_with_plan(
    query,
    index,
    plan,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

That means you can inspect:

- the approximate candidate set
- the exact rerank stage
- the query and document vector counts consumed by each stage

## Why Reranking Alone Is Not Enough

The talk's strongest retrieval point is worth making explicit in the docs:

if stage 1 never retrieves the right candidates, stage 2 cannot save you.

That sounds obvious, but it is the reason Kayak exposes candidate generation,
candidate windows, and stage profiles directly instead of hiding them.

For example, if your approximate stage produces very low recall at
`candidate_k=1000`, no exact reranker can reconstruct the documents that were
never surfaced in the first place.

That is why Kayak treats candidate generation and exact reranking as separate,
inspectable stages.

## The Retrieval Constraint Is Sublinear Search

Late interaction only becomes a retrieval paradigm, rather than just an exact
scoring function, when you can combine local interaction with sublinear
candidate retrieval.

That is the key practical distinction:

- exact scoring tells you what the rich matching function is
- staged retrieval tells you how to use that matching function without touching
  every document on every query

Kayak's search-plan API exists to keep that boundary explicit.

## ColBERT-Style 128-Dim Embeddings

The Mojo path is intended for ColBERT-style embeddings, where vectors are
128-dimensional and queries and documents each carry multiple token vectors.

That is the shape where:

- `flat_dim128` and `hybrid_flat_dim128` layouts make sense
- explicit late-interaction benchmarking is meaningful
- the Mojo exact CPU backend is the path to optimize and measure

If you are still sketching logic with tiny toy arrays, use the NumPy reference
path. Use 128-dimensional examples when you want to exercise the real Mojo
workflow.
