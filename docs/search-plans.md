# Search Plans

The central idea in kayak: retrieval as an explicit, inspectable pipeline.

---

## Why search plans exist

Late interaction retrieval has multiple stages with real trade-offs at each one:

- **Stage 1** generates candidates — fast and approximate, or slow and exact
- **Stage 2** reranks them with exact MaxSim over the candidate window
- **Stage 3** optionally verifies with text-level refinement

Every choice affects latency, quality, and memory. Kayak makes those choices explicit as a Python object you build, run, and inspect.

---

## Building a plan

```python
import kayak

plan = kayak.document_proxy_search_plan(
    final_k=10,
    candidate_k=100,
    query_vector_budget=32,
    document_vector_budget=64,
)
```

The plan is a plain Python dataclass:

```python
SearchPlan(
    candidate_generator=document_proxy(
        query_vector_budget=32,
        document_vector_budget=64,
    ),
    final_k=10,
    candidate_k=100,
    stage2_reference_operator=exact_late_interaction,
    stage3_verifier=none,
)
```

Nothing runs until you call `search_with_plan`. The plan is just a description.

---

## Running a plan

```python
result = kayak.search_with_plan(query, index, plan)
```

---

## Inspecting the result

`SearchPlanResult` carries the full intermediate state of every stage.

### Final results

```python
result.hits
# (SearchHit(doc_id='doc-a', score=1.92),
#  SearchHit(doc_id='doc-c', score=1.87), ...)
```

### Stage 1 — what the candidate generator returned

```python
result.candidate_stage.hits
# 100 candidates with approximate scores
```

### Stage 2 — the exact reranking

```python
result.stage2_reference.stage_name             # 'exact_late_interaction'
result.stage2_reference.input_hit_count        # 100 — candidates entering stage 2
result.stage2_reference.output_hit_count       # 10  — final results
result.stage2_reference.query_vector_count     # 32  — query vectors used
result.stage2_reference.document_vector_count  # 6400 — doc vectors scored (100 × 64)
```

### Compare stage 1 and stage 2

```python
stage1_ids = {hit.doc_id for hit in result.candidate_stage.hits[:10]}
stage2_ids = {hit.doc_id for hit in result.hits}

dropped   = stage1_ids - stage2_ids  # docs the reranker removed
promoted  = stage2_ids - stage1_ids  # docs the reranker surfaced
```

This is how you verify that your Stage 1 approximation is actually finding the right candidates.

---

## Stage 1 options

### `exact_full_scan`

Scores every document with exact MaxSim. No approximation. Correct by definition.

```python
plan = kayak.exact_full_scan_search_plan(final_k=10)
```

Use this as your correctness baseline. Slow on large corpora, but the right answer.

### `document_proxy`

Approximate candidate generation using per-document vector averages. Fast — skips most documents before exact scoring.

```python
plan = kayak.document_proxy_search_plan(
    final_k=10,
    candidate_k=100,
    query_vector_budget=32,
    document_vector_budget=64,
)
```

`candidate_k` controls the trade-off: larger windows recover more of the exact results but cost more in Stage 2.

`query_vector_budget` and `document_vector_budget` cap how many vectors the proxy uses — set to 0 to use all vectors.

---

## Stage 2 options

### `exact_late_interaction`

Full MaxSim reranking over the candidate window. Default for approximate Stage 1.

### `noop_topk`

No reranking — take the top-k directly from Stage 1 scores. Default for `exact_full_scan` since Stage 1 is already exact.

---

## Stage 3 — text verification

Attach query text and document texts to enable text-aware refinement after exact reranking:

```python
query = kayak.query(vectors, text="founded in 1984 in a church")

docs = kayak.documents(
    ["doc-a", "doc-b"],
    [vectors_a, vectors_b],
    texts=[
        "Zama Dance School was founded in 1984 in a church.",
        "Lyon is a city in France.",
    ],
)
index = docs.pack()

plan = kayak.exact_full_scan_search_plan(
    final_k=5,
    candidate_k=20,
    stage3_verifier=kayak.clause_text_stage3_verifier_operator(),
)

result = kayak.search_with_plan(query, index, plan)

result.stage3_verifier.stage_name           # 'clause_text'
result.stage3_verifier.document_text_count  # 20 — texts evaluated
```

Text is only attached when a text-family operator needs it. Kayak does not hide text behind generic tensors.

---

## Measuring faithfulness

How many of your approximate Stage 1 candidates match the exact full scan?

```python
exact_plan  = kayak.exact_full_scan_search_plan(final_k=10)
approx_plan = kayak.document_proxy_search_plan(final_k=10, candidate_k=100)

exact_result  = kayak.search_with_plan(query, index, exact_plan)
approx_result = kayak.search_with_plan(query, index, approx_plan)

exact_ids  = {hit.doc_id for hit in exact_result.hits}
approx_ids = {hit.doc_id for hit in approx_result.hits}

recall = len(exact_ids & approx_ids) / len(exact_ids)
print(f"Stage 1 recall: {recall:.0%}")
```

This is how you tune `candidate_k` — increase it until recall is acceptable, then check Stage 2 latency against the cost you can afford.
