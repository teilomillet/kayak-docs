# Quickstart

This is the shortest verified path to the Mojo backend from Python.

It uses ColBERT-style 128-dimensional vectors because that is the shape the
Mojo path is designed for.

## One File, Mojo First

```python
import numpy as np
import kayak


def dim128(index: int) -> np.ndarray:
    vector = np.zeros(128, dtype=np.float32)
    vector[index] = 1.0
    return vector


BACKEND = kayak.MOJO_EXACT_CPU_BACKEND

query = kayak.query(np.stack([dim128(0), dim128(1)]))
documents = kayak.documents(
    ["doc-a", "doc-b"],
    [
        np.stack([dim128(0), dim128(1)]),
        np.stack([dim128(0), dim128(0)]),
    ],
)
index = documents.pack()

hits = kayak.search(query, index, k=2, backend=BACKEND)
scores = kayak.maxsim(query, index, backend=BACKEND)

print("hits:", [(hit.doc_id, hit.score) for hit in hits])
print("scores:", scores.numpy().tolist())
```

That is the core flow:

1. Build a `LateQuery`.
2. Build `LateDocuments`.
3. Pack them into a `LateIndex`.
4. Pass `backend=kayak.MOJO_EXACT_CPU_BACKEND`.

## Why The `BACKEND` Variable Matters

Installing Pixi plus Mojo makes the Mojo backend available.
It does not make the SDK silently switch defaults.

The `BACKEND` constant above is the recommended pattern when you want your own
application code to behave as "use Mojo unless I override it."

## A Slightly More Explicit 128-Dim Layout

If you want the query and index layouts to be explicit too, convert them:

```python
flat_query = query.to_layout("flat_dim128")
hybrid_index = index.to_layout("hybrid_flat_dim128")

scores = kayak.maxsim(
    flat_query,
    hybrid_index,
    backend=BACKEND,
)
```

Use that form when you want to make the 128-dimensional flattened layout part
of the code you reason about, benchmark, or profile.

## Batch Search Against The Same Index

If you have many queries for the same index, use the batch API:

```python
batch = kayak.query_batch(
    [
        np.stack([dim128(0), dim128(1)]),
        np.stack([dim128(0), dim128(1), dim128(2)]),
    ]
)

hits_by_query = kayak.search_batch(
    batch,
    index,
    k=2,
    backend=BACKEND,
)
```

That is one of the main reasons to install Mojo in the first place.

## Next Steps

- [Usage Patterns](usage-patterns.md) for the shortest map from task to API
- [real-usage-with-mojo.ipynb](notebooks/real-usage-with-mojo.ipynb) for a complete executed example
- [Using the Mojo Backend](mojo-backend.md) for when to use each layout and API
- [Late Interaction](concepts.md) for the scoring model and explicit vector budgets
- [Search Plans](search-plans.md) for approximate stage 1 plus exact stage 2 pipelines
