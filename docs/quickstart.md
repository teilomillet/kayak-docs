# Quickstart

A working late-interaction search in under 20 lines.

---

## 1. Create a query

A query is a matrix of token vectors — one row per token.

```python
import kayak
import numpy as np

query = kayak.query(
    np.array(
        [[1.0, 0.0],
         [0.0, 1.0]],
        dtype=np.float32,
    )
)
# 2 query tokens, 2-dimensional embeddings
```

---

## 2. Create documents

Each document has its own number of token vectors — no padding required.

```python
docs = kayak.documents(
    ["doc-a", "doc-b", "doc-c"],
    [
        np.array([[1.0, 0.0], [0.0, 1.0]], dtype=np.float32),
        np.array([[1.0, 0.0], [0.5, 0.5]], dtype=np.float32),
        np.array([[0.2, 0.8], [0.8, 0.2]], dtype=np.float32),
    ],
)
```

---

## 3. Pack into an index

```python
index = docs.pack()
```

---

## 4. Search

```python
hits = kayak.search(query, index, k=2)

for hit in hits:
    print(hit.doc_id, hit.score)
# doc-a  2.0
# doc-b  1.75
```

---

## 5. Score all documents

```python
scores = kayak.maxsim(query, index)

print(scores.doc_ids)   # ('doc-a', 'doc-b', 'doc-c')
print(scores.values)    # array([2.0, 1.75, 1.6], dtype=float32)
```

---

## Next steps

- [Understand late interaction](concepts.md) — what MaxSim computes and why vector counts matter
- [Use search plans](search-plans.md) — explicit multi-stage pipelines you can inspect and tune
