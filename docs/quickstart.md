# Quickstart

The shortest Python SDK path after installation.

## Recommendation First

Use this decision rule unless you have a specific reason not to:

- start with `kayak.open_text_retriever(...)` if your inputs are raw text
- start with `kayak.query(...)` plus `kayak.documents(...).pack()` if you already own token vectors
- if many queries hit the same data, reuse one loaded or packed index and prefer `kayak.search_batch(...)`
- if `kayak.MOJO_EXACT_CPU_BACKEND` is available, pass it explicitly for repeated-query exact search

Current low-level fast path, as implemented:

- `kayak.search(...)` or `kayak.search_batch(...)`
- `backend=kayak.MOJO_EXACT_CPU_BACKEND`
- packed index layout
- nested query layout

That means the practical optimization target is index reuse plus backend selection, not making users own backend-specific primitives in application code.

## One File

```python
import numpy as np
import kayak


def dim128(index: int) -> np.ndarray:
    vector = np.zeros(128, dtype=np.float32)
    vector[index] = 1.0
    return vector


query = kayak.query(np.stack([dim128(0), dim128(1)]))
documents = kayak.documents(
    ["doc-a", "doc-b"],
    [
        np.stack([dim128(0), dim128(1)]),
        np.stack([dim128(0), dim128(0)]),
    ],
)
index = documents.pack()

hits = kayak.search(query, index, k=2)
scores = kayak.maxsim(query, index)

print("hits:", [(hit.doc_id, hit.score) for hit in hits])
print("scores:", scores.numpy().tolist())
```

1. `kayak.query(...)` wraps one token matrix as a `LateQuery`.
2. `kayak.documents(...)` collects aligned ids and token matrices.
3. `.pack()` materializes the search-ready `LateIndex`.
4. `kayak.search(...)` and `kayak.maxsim(...)` run exact scoring.

If the current environment exposes additional backends, you can pass `backend=...` explicitly.

## If You Start From Text Instead Of Vectors

```python
retriever = kayak.open_text_retriever(
    encoder="colbert",
    store="kayak",
    encoder_kwargs={"model_name": "colbert-ir/colbertv2.0"},
    store_kwargs={"path": "./kayak-index"},
)

retriever.upsert_texts(doc_ids, texts)
hits = retriever.search_text(query_text, k=10)
```

## Make The Layout Explicit When You Care About It

```python
flat_query = query.to_layout("flat_dim128")
hybrid_index = index.to_layout("hybrid_flat_dim128")

scores = kayak.maxsim(flat_query, hybrid_index)
```

Use this form when you want the layout choice to be visible in code and measurements.

## Batch Search On The Same Index

```python
batch = kayak.query_batch(
    [
        np.stack([dim128(0), dim128(1)]),
        np.stack([dim128(0), dim128(1), dim128(2)]),
    ]
)

hits_by_query = kayak.search_batch(batch, index, k=2)
```

Use this when many queries target the same already-built index. This is the main public fast path for repeated traffic.

## Open The Next Page Based On What You Need

| If you want to... | Open... |
| --- | --- |
| choose the long-term API shape | [Usage Patterns](usage-patterns.md) |
| pass a Hugging Face ColBERT checkpoint or your own model | [Text Encoders](text-encoders.md) |
| open a full executed walkthrough | [full local SDK walkthrough](notebooks/real-usage-with-mojo.ipynb) |
| keep an existing database for storage | [Storage + Search](storage-and-search.md) |
