# Using The Mojo Backend

This page is about one specific goal:

"I want my Python Kayak setup to use the Mojo exact CPU backend as my normal
path, not accidentally fall back to NumPy because I only installed the package."

## What The Mojo Backend Actually Gives You

`kayak.MOJO_EXACT_CPU_BACKEND` is the exact CPU execution path for:

- `kayak.maxsim(...)`
- `kayak.maxsim_batch(...)`
- `kayak.search(...)`
- `kayak.search_batch(...)`
- `kayak.search_with_plan(...)`

Use it for:

- exact local scoring
- exact local top-k search
- exact reranking in stage-aware search plans
- repeated and batched queries against the same index

It does not change the retrieval semantics. It changes the executor.

That means:

- the scores should match the NumPy reference path on supported inputs
- the stage structure stays the same
- the backend choice is still explicit at the call site

## Make Mojo Available

The requirement is not Pixi itself. The requirement is that Kayak can invoke a
usable `mojo` command.

Two verified install stories are:

- put Python 3.11, Mojo, and Kayak in the same Pixi environment
- or install `kayak` with UV or pip in an environment where `mojo` is already available

In both cases:

1. Verify `kayak.available_backends()` includes `mojo_exact_cpu`.
2. Define a backend constant in your application code.

```bash
pixi add python=3.11 "mojo>=0.26.3.0.dev2026041020,<0.27"
pixi add --pypi kayak
```

```python
import kayak

print(kayak.available_backends())
# ('numpy_reference', 'mojo_exact_cpu')
```

If that tuple does not include `mojo_exact_cpu`, stop and inspect
`kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND)` before assuming anything.

UV works too if Mojo is already installed first and is discoverable:

```bash
mojo --version
uv add kayak
uv run python - <<'PY'
import kayak
print(kayak.available_backends())
print(kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND).availability_reason)
PY
```

That is equally valid as long as the `mojo` CLI is actually visible to Kayak.

## How To Make Your Own Code Default To Mojo

Low-level Kayak calls do not auto-select Mojo. If you want "default to Mojo"
behavior in your own codebase, define that default yourself:

```python
import kayak

BACKEND = kayak.MOJO_EXACT_CPU_BACKEND
```

Then thread it through the operations you care about:

```python
scores = kayak.maxsim(query, index, backend=BACKEND)
hits = kayak.search(query, index, k=10, backend=BACKEND)
hits_by_query = kayak.search_batch(batch, index, k=10, backend=BACKEND)
result = kayak.search_with_plan(query, index, plan, backend=BACKEND)
```

That is better than relying on hidden environment behavior because:

- it is explicit in code review
- tests can override it easily
- benchmarks stay honest about which executor ran

If you want the highest-level text workflow to do that selection for you,
`open_text_retriever(...)` already prefers Mojo automatically when it is
available.

## Use 128-Dim Embeddings For The Mojo Path

The validated Mojo workflow is ColBERT-style 128-dimensional embeddings.

That is the right shape for:

- `query.to_layout("flat_dim128")`
- `index.to_layout("hybrid_flat_dim128")`
- the prepared packed-index fast path

If you are sketching with tiny 2D toy vectors, keep those examples on the
NumPy backend. Switch to 128-dimensional vectors before you judge the Mojo
path.

## Fastest Normal Usage Pattern

For repeated exact search against one index, use this pattern:

```python
import numpy as np
import kayak


def dim128(index: int) -> np.ndarray:
    vector = np.zeros(128, dtype=np.float32)
    vector[index] = 1.0
    return vector


BACKEND = kayak.MOJO_EXACT_CPU_BACKEND

query = kayak.query(np.stack([dim128(0), dim128(1)]))
index = kayak.documents(
    ["doc-a", "doc-b"],
    [
        np.stack([dim128(0), dim128(1)]),
        np.stack([dim128(0), dim128(0)]),
    ],
).pack()

hits = kayak.search(query, index, k=2, backend=BACKEND)
```

Reasons this is the right baseline:

- `documents.pack()` gives you a reusable `LateIndex`
- reusing the same `LateIndex` instance lets Kayak reuse prepared index state
- `search(...)` gives direct exact top-k output instead of forcing a full score vector first

## When To Use Explicit Flat Layouts

If you want the 128-dimensional layout transformation to be visible in the code,
convert it explicitly:

```python
flat_query = query.to_layout("flat_dim128")
hybrid_index = index.to_layout("hybrid_flat_dim128")

scores = kayak.maxsim(
    flat_query,
    hybrid_index,
    backend=BACKEND,
)
```

Choose this form when:

- you want profiling boundaries to line up with the flat layout
- you want layout choice to be part of the benchmark setup
- you are reasoning about storage and conversion cost explicitly

## Use Batch APIs For Repeated Query Traffic

If the index is fixed and queries are the thing changing, use the batch API.

```python
batch = kayak.query_batch(
    [
        np.stack([dim128(0), dim128(1)]),
        np.stack([dim128(0), dim128(1), dim128(2)]),
    ]
)

scores_by_query = kayak.maxsim_batch(
    batch,
    index,
    backend=BACKEND,
)

hits_by_query = kayak.search_batch(
    batch,
    index,
    k=10,
    backend=BACKEND,
)
```

This is a better fit than looping one query at a time in Python when you are
running repeated local evaluation or benchmark workloads.

## Use Mojo Inside Search Plans

Search plans are explicit stage pipelines. The backend only affects the stages
that perform exact late-interaction scoring.

### Exact full scan

```python
plan = kayak.exact_full_scan_search_plan(final_k=10)
result = kayak.search_with_plan(query, index, plan, backend=BACKEND)
```

Here stage 1 is already exact, so stage 2 defaults to `noop_topk`.

### Approximate candidate generation plus exact Mojo rerank

```python
plan = kayak.document_proxy_search_plan(
    final_k=10,
    candidate_k=100,
    query_vector_budget=32,
    document_vector_budget=64,
)

result = kayak.search_with_plan(query, index, plan, backend=BACKEND)
```

In this plan:

- stage 1 uses the document-proxy candidate generator
- stage 2 uses exact late interaction on the candidate window
- the exact rerank stage is where the backend matters most

That is the standard pattern when you want a fast approximate candidate stage
followed by exact scoring on a smaller set.

## What To Do With It

Use the Mojo backend when you need to answer one of these questions with exact
local execution:

- "What are the exact top-k results for this query and index?"
- "Did my approximate candidate generator recover the exact winners?"
- "How much exact stage-2 work did this candidate window create?"
- "What is steady-state exact search throughput on a fixed local index?"

The point is not just "make search faster." The point is "keep the exact path
programmable and measurable from Python."

## Troubleshooting

### `available_backends()` only shows `numpy_reference`

Check:

```python
info = kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND)
print(info.available)
print(info.availability_reason)
```

Most often this means one of:

- Mojo is not installed in the active environment
- `mojo` is not on `PATH`
- the wrong `mojo` binary is being picked up

If you need to pin the binary, set `KAYAK_MOJO_CLI`.

`KAYAK_MOJO_CLI` can be either a binary path or a full command prefix such as:

```bash
export KAYAK_MOJO_CLI="bash /full/path/to/run_mojo_with_wrapper.sh"
```

### Mojo reports a bundled backend/compiler mismatch

That means the installed `kayak` wheel was built with a different Mojo release
than the one you are invoking locally. Install matching `kayak` and `mojo`
versions, then rerun the same code.

### The first Mojo call is slow

That is expected. The first call may build backend artifacts and prepare index
state. Measure steady-state after warmup, not only the first invocation.

### You want Mojo to be implicit everywhere

The SDK does not do that. Make it explicit in your own code with a shared
`BACKEND = kayak.MOJO_EXACT_CPU_BACKEND` constant or a small wrapper layer.
