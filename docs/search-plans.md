# Search Plans

Search plans are Kayak's way of making retrieval pipelines explicit and
inspectable from Python.

The important thing for Python users is this:

- stage structure is explicit
- candidate generation is explicit
- exact reranking is explicit
- backend choice is explicit

That makes search plans the right place to reason about both quality and cost.

## Choose The Plan Task

=== "I Need The Exact Baseline"

    Start with:

    - `kayak.exact_full_scan_search_plan(...)`
    - [Exact Baseline Plan](#exact-baseline-plan)

=== "I Need Candidate Stage Plus Exact Rerank"

    Start with:

    - `kayak.document_proxy_search_plan(...)`
    - [Approximate Candidate Stage Plus Exact Rerank](#approximate-candidate-stage-plus-exact-rerank)

=== "I Need To Check Whether Stage 1 Is Good Enough"

    Jump to:

    - [Comparing Approximate And Exact Results](#comparing-approximate-and-exact-results)
    - [Candidate Generation Knobs](#candidate-generation-knobs)

=== "I Need Text Verification After Rerank"

    Jump to:

    - [Stage 3 Text Verification](#stage-3-text-verification)

## A Plan Is A Description, Not Execution

```python
plan = kayak.document_proxy_search_plan(
    final_k=10,
    candidate_k=100,
    query_vector_budget=32,
    document_vector_budget=64,
)
```

Nothing runs yet.

Execution starts when you call:

```python
result = kayak.search_with_plan(
    query,
    index,
    plan,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

That `backend=` argument is where you choose the exact executor for the plan.

## What The Backend Changes Inside A Plan

The backend matters only for the stages that perform exact late-interaction
scoring.

### `exact_full_scan_search_plan(...)`

Stage 1 is already exact, so the plan defaults to:

- exact full-scan candidate generation
- `noop_topk` in stage 2

When you run this plan, the exact full scan is what uses the selected backend.

### `document_proxy_search_plan(...)`

Stage 1 is approximate document-proxy candidate generation.
Stage 2 is exact late-interaction reranking.

When you run this plan:

- stage 1 still does the approximate proxy scoring
- stage 2 exact reranking uses the chosen backend

That is usually the most important plan to run when comparing candidate stages against exact reranking.

## Why These Plans Matter

The talk makes a useful point that fits directly here:

retrieve-and-rerank is only a good abstraction if the candidate generator
actually reaches the relevant documents.

That means the real question is not only:

- "How strong is my reranker?"

It is also:

- "What recall did my candidate stage achieve before reranking even began?"

Kayak exposes both because late interaction is most useful when you can inspect
that boundary directly.

## Exact Baseline Plan

Use this as the correctness baseline:

```python
exact_plan = kayak.exact_full_scan_search_plan(final_k=10)

exact_result = kayak.search_with_plan(
    query,
    index,
    exact_plan,
    backend=kayak.NUMPY_REFERENCE_BACKEND,
)
```

This answers:

- What are the exact winners?
- What is the exact full-scan cost?

## Approximate Candidate Stage Plus Exact Rerank

Use this when you want a realistic retrieval pipeline:

```python
approx_plan = kayak.document_proxy_search_plan(
    final_k=10,
    candidate_k=100,
    query_vector_budget=32,
    document_vector_budget=64,
)

approx_result = kayak.search_with_plan(
    query,
    index,
    approx_plan,
    backend=kayak.NUMPY_REFERENCE_BACKEND,
)
```

This answers:

- Did stage 1 find the right candidate window?
- How much exact stage-2 work did that candidate window create?
- What trade-off did `candidate_k` buy me?

This is the plan shape to use when you want to test whether a cheaper stage 1
is still producing a candidate window that makes exact reranking worthwhile.

## Inspecting The Result

`SearchPlanResult` exposes the intermediate state directly.

```python
result.hits
result.candidate_stage.hits
result.stage2_reference
result.stage3_verifier
```

The most important profiling fields are:

```python
result.stage2_reference.stage_name
result.stage2_reference.input_hit_count
result.stage2_reference.output_hit_count
result.stage2_reference.query_vector_count
result.stage2_reference.document_vector_count
```

Those tell you how much exact work the plan actually executed.

## Comparing Approximate And Exact Results

This is the simplest faithfulness check:

```python
exact_plan = kayak.exact_full_scan_search_plan(final_k=10)
approx_plan = kayak.document_proxy_search_plan(final_k=10, candidate_k=100)

exact_result = kayak.search_with_plan(
    query,
    index,
    exact_plan,
    backend=kayak.NUMPY_REFERENCE_BACKEND,
)
approx_result = kayak.search_with_plan(
    query,
    index,
    approx_plan,
    backend=kayak.NUMPY_REFERENCE_BACKEND,
)

exact_ids = {hit.doc_id for hit in exact_result.hits}
approx_ids = {hit.doc_id for hit in approx_result.hits}

recall = len(exact_ids & approx_ids) / len(exact_ids)
```

Use that before making claims about a candidate generator.

If recall is poor here, the right fix is usually not "use a stronger stage-2
reranker." The right fix is usually to widen or improve stage 1.

## Candidate Generation Knobs

`document_proxy_search_plan(...)` exposes the main stage-1 controls:

- `candidate_k`
- `query_vector_budget`
- `document_vector_budget`

`candidate_k` is the main recall-vs-cost knob.

The vector budgets control how many vectors the proxy stage averages before
producing candidate scores:

- `0` means "use all available vectors"
- smaller values reduce proxy-stage work
- smaller values can also reduce candidate quality

The important implication is that these budgets do not just tune latency. They
also change what evidence stage 1 can even see.

That is why Kayak keeps the budgets explicit in the plan object instead of
hiding them in backend-specific config.

## Stage 3 Text Verification

You can add a text-family verifier after exact reranking:

```python
query = kayak.query(
    query_vectors,
    text="founded in 1984 in a church artistic director",
)

documents = kayak.documents(
    ["doc-context", "doc-answer"],
    [doc_vectors_a, doc_vectors_b],
    texts=[
        "Gugulethu township logo emblem heritage schools history",
        "Zama Dance School was founded in 1984 in a church and the longest serving employee is the artistic director.",
    ],
)
index = documents.pack()

plan = kayak.exact_full_scan_search_plan(
    final_k=1,
    candidate_k=2,
    stage3_verifier=kayak.clause_text_stage3_verifier_operator(),
)

result = kayak.search_with_plan(
    query,
    index,
    plan,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

This requires:

- `query.text`
- `documents(..., texts=...)`

The verifier is explicit because text materialization has real cost too.

## What To Tune First

If you are starting from scratch:

1. Run an exact full-scan baseline.
2. Run a document-proxy plan with a generous `candidate_k`.
3. Measure overlap with the exact winners.
4. Reduce or increase `candidate_k` based on recall and exact stage-2 cost.
5. Keep the backend fixed while you tune the plan.

That sequence gives you a defensible tuning workflow instead of anecdotal
"seems good enough" iteration.

## Read Next

- [Usage Patterns](usage-patterns.md) when you want the shorter task-to-API version
- [Late Interaction](concepts.md) when you want the underlying cost model
- [API Reference](api.md) when you want the exact constructor and return types
