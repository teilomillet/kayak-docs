# Kayak Python SDK

Kayak is a local Python SDK for late-interaction retrieval.

It keeps the parts that matter explicit:

- query vector count
- document vector count
- index layout
- backend choice
- candidate window size

That is the point of the SDK. You should be able to see what the retrieval
pipeline is doing, measure it, and change it deliberately.

## If You Want Mojo, Start With Pixi

If your goal is to use the Mojo backend, do not stop at:

```bash
uv add kayak
```

That installs the Python package, but not the Mojo CLI. In that setup Kayak
will expose only the NumPy reference backend.

The verified path for the Mojo backend is:

1. Create or use a Pixi environment that includes Mojo.
2. Install `kayak` into that same environment.
3. Pass `backend=kayak.MOJO_EXACT_CPU_BACKEND` on the calls you want on Mojo.

Use the [installation guide](installation.md) for the exact workflow.

## What The Mojo Backend Is For

`kayak.MOJO_EXACT_CPU_BACKEND` is the exact CPU scorer for the Python SDK.

Use it when you want:

- exact local MaxSim scoring on ColBERT-style embeddings
- exact top-k search from Python
- exact reranking inside explicit search plans
- repeated queries against the same index, including batch search

It does not silently replace the default backend. Kayak keeps backend choice
explicit on purpose.

## Minimal Mojo Example

The validated Mojo path is ColBERT-style 128-dimensional embeddings.

```python
import numpy as np
import kayak


def dim128(index: int) -> np.ndarray:
    vector = np.zeros(128, dtype=np.float32)
    vector[index] = 1.0
    return vector


backend = kayak.MOJO_EXACT_CPU_BACKEND

query = kayak.query(np.stack([dim128(0), dim128(1)]))
index = kayak.documents(
    ["doc-a", "doc-b"],
    [
        np.stack([dim128(0), dim128(1)]),
        np.stack([dim128(0), dim128(0)]),
    ],
).pack()

hits = kayak.search(query, index, k=2, backend=backend)
scores = kayak.maxsim(query, index, backend=backend)
```

The important pattern is the `backend` variable. That is how you make your own
application code default to Mojo, even though the SDK itself does not switch
defaults automatically.

## Why Kayak Is Different

Most vector search APIs hide the retrieval pipeline behind a single method call.
Kayak does not.

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

result.hits
result.candidate_stage.hits
result.stage2_reference.query_vector_count
result.stage2_reference.document_vector_count
```

That makes it practical to answer questions like:

- How many query vectors did stage 2 really score?
- How many document vectors did the candidate window force me to touch?
- Did my approximate candidate generator preserve the exact top-k?

## Read Next

- [Installation](installation.md) for the Pixi plus Mojo setup
- [Quickstart](quickstart.md) for a working Mojo-first script
- [Usage Patterns](usage-patterns.md) for which API to call for each task
- [Storage + Search](storage-and-search.md) for the vector-db-as-storage, Kayak-as-search-engine deployment shape
- [Using the Mojo Backend](mojo-backend.md) for how to make Mojo your normal code path
- [Vector Database Integrations](vector-databases.md) for LanceDB, Qdrant, Weaviate, ChromaDB, and generic stage-1/stage-2 handoff patterns
- [Late Interaction](concepts.md) for the scoring model and explicit vector budgets
- [Search Plans](search-plans.md) for approximate stage 1 plus exact stage 2 pipelines
