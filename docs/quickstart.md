# Quickstart

The shortest verified path once installation is correct. Uses ColBERT-style 128-dimensional vectors — the shape the Mojo exact path is designed for.

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

1. `kayak.query(...)` wraps one token matrix as a `LateQuery`.
2. `kayak.documents(...)` collects aligned ids and token matrices.
3. `.pack()` materializes the search-ready `LateIndex`.
4. `kayak.search(...)` and `kayak.maxsim(...)` run exact scoring on the selected backend.

## If You Start From Text Instead Of Vectors

Use one encoder plus one retriever:

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

For text workflows, `open_text_retriever(...)` already prefers the Mojo backend
automatically when the active environment can actually run it.

## Make The Layout Explicit When You Care About It

If you want the query and index layouts to be part of the code you benchmark or
profile, convert them explicitly:

```python
flat_query = query.to_layout("flat_dim128")
hybrid_index = index.to_layout("hybrid_flat_dim128")

scores = kayak.maxsim(
    flat_query,
    hybrid_index,
    backend=BACKEND,
)
```

Use this form when you want the 128-dimensional flattened layout to be visible
in your code and measurements.

## Batch Search On The Same Index

If many queries hit the same index, use the batch API:

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

That is one of the main reasons to install Mojo correctly in the first place.

## Open The Next Page Based On What You Need

| If you want to... | Open... |
| --- | --- |
| choose the long-term API shape | [Usage Patterns](usage-patterns.md) |
| pass a Hugging Face ColBERT checkpoint or your own model | [Text Encoders](text-encoders.md) |
| open a full executed walkthrough | [real-usage-with-mojo.ipynb](notebooks/real-usage-with-mojo.ipynb) |
| understand the backend and layout surface | [Mojo Backend](mojo-backend.md) |
| keep an existing database for storage | [Storage + Search](storage-and-search.md) |
