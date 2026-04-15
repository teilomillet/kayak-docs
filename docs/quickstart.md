<div class="kayak-hero kayak-hero--compact" markdown>
<div class="kayak-hero__main" markdown>

<p class="kayak-eyebrow">Shortest working example</p>

# Run one exact search on the Mojo backend from Python

<p class="kayak-lead">
This is the shortest verified path once installation is correct. It uses
ColBERT-style 128-dimensional vectors because that is the shape the Mojo exact
path is designed for.
</p>

<div class="kayak-action-row" markdown>

[Installation](installation.md){ .md-button .md-button--primary }
[Usage Patterns](usage-patterns.md){ .md-button }
[Open notebook](notebooks/real-usage-with-mojo.ipynb){ .md-button }

</div>

</div>
<aside class="kayak-hero__aside" markdown>

<ul class="kayak-link-list">
  <li><strong>Inputs</strong> one query, one document set, 128-dimensional token vectors</li>
  <li><strong>Core APIs</strong> <code>kayak.query(...)</code>, <code>kayak.documents(...).pack()</code>, <code>kayak.search(...)</code></li>
  <li><strong>Backend</strong> pass <code>backend=kayak.MOJO_EXACT_CPU_BACKEND</code> explicitly on low-level calls</li>
</ul>

</aside>
</div>

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

## What That Script Does

<div class="kayak-card-grid" markdown>

<section class="kayak-card" markdown>
### 1. Build the query

<code>kayak.query(...)</code> wraps one token matrix as a <code>LateQuery</code>.
</section>

<section class="kayak-card" markdown>
### 2. Build the documents

<code>kayak.documents(...)</code> collects aligned ids and token matrices.
</section>

<section class="kayak-card" markdown>
### 3. Pack the index

<code>.pack()</code> materializes the search-ready <code>LateIndex</code>.
</section>

<section class="kayak-card" markdown>
### 4. Search explicitly

<code>kayak.search(...)</code> and <code>kayak.maxsim(...)</code> run exact scoring on the selected
backend.
</section>

</div>

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
