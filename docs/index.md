# Kayak Python SDK

<div class="kayak-hero" markdown>

<p class="kayak-eyebrow">Late-interaction retrieval for Python</p>

# Kayak Python SDK

<p class="kayak-lead">
Use the Mojo backend when you want exact local late-interaction search, keep
your current vector database when you need it, and keep the retrieval pipeline
explicit enough to measure.
</p>

[Start Here](start-here.md){ .md-button .md-button--primary }
[Installation](installation.md){ .md-button }
[Examples](examples.md){ .md-button }

<div class="kayak-pill-row" markdown>

- `Mojo exact CPU backend`
- `ColBERT-style 128-d embeddings`
- `Batch search on one loaded slice`
- `Keep LanceDB / PgVector / Qdrant / Weaviate / Chroma`

</div>

</div>

Kayak is a local Python SDK for late-interaction retrieval.

It keeps the parts that matter explicit:

- query vector count
- document vector count
- index layout
- backend choice
- candidate window size

That is the point of the SDK. You should be able to see what the retrieval
pipeline is doing, measure it, and change it deliberately.

<div class="grid cards" markdown>

- __Choose your path first__

  Use this if you are still thinking in terms like "I have text", "I have a
  vector DB", or "I need repeated-query speed" and do not want to translate
  that into Kayak terms yet.

  [Open start-here guide](start-here.md)

- __Install and get one working Mojo setup__

  Start here if your goal is simply "make `import kayak` use the Mojo backend
  correctly."

  [Open installation guide](installation.md)

- __Choose the right API shape__

  Use this when you want to know whether you should open a retriever, load one
  reusable index, call `search_batch(...)`, or move to a search plan.

  [Open usage patterns](usage-patterns.md)

- __Start from text or your own model__

  Use the built-in ColBERT encoder for ColBERT checkpoints, or wrap your own
  model with the callable encoder.

  [Open text encoders](text-encoders.md)

- __Keep your existing database__

  Use Kayak as the search engine while LanceDB, PgVector, Qdrant, Weaviate, or
  Chroma stay responsible for persistence.

  [Open storage and integrations](storage-and-search.md)

- __Open a real executed example__

  Use this when you want a notebook or script, not another conceptual page.

  [Open examples](examples.md)

- __Look up exact APIs__

  Use this when you already know the path and just want the stable Python
  surface.

  [Open API reference](api.md)

</div>

## Start From Your Situation

=== "I Want One Working Search"

    Start with:

    - [Start Here](start-here.md)
    - [Installation](installation.md)
    - [Quickstart](quickstart.md)

=== "I Start From Text"

    Start with:

    - [Start Here](start-here.md)
    - [Text Encoders](text-encoders.md)
    - [Usage Patterns](usage-patterns.md)

=== "I Already Have A Vector Database"

    Start with:

    - [Start Here](start-here.md)
    - [Storage + Search](storage-and-search.md)
    - [Vector Databases](vector-databases.md)

=== "I Need Throughput Or Concurrency"

    Start with:

    - [Usage Patterns](usage-patterns.md)
    - [Using the Mojo Backend](mojo-backend.md)
    - [Hosted Engine Python](hosted-engine-python.md) if your target is the hosted snapshot runtime

## The Shortest Useful Journey

<div class="grid cards" markdown>

- __Install Mojo correctly__

  Make sure Kayak can actually discover a usable `mojo` CLI.

  [Open installation](installation.md)

- __Run one exact search__

  Use the shortest working ColBERT-style 128-d example.

  [Open quickstart](quickstart.md)

- __Choose the right API shape__

  Move from one working search to the right long-term workflow.

  [Open usage patterns](usage-patterns.md)

</div>

## The Main User Paths

Most users fall into one of these shapes:

| If you want to... | Start here |
| --- | --- |
| choose by your current situation instead of by API name | [Start Here](start-here.md) |
| get one Mojo-capable install working | [Installation](installation.md) |
| run one local exact search quickly | [Quickstart](quickstart.md) |
| choose between retriever, batch search, and plans | [Usage Patterns](usage-patterns.md) |
| pass a Hugging Face ColBERT checkpoint or your own model | [Text Encoders](text-encoders.md) |
| keep your vector database and let Kayak search | [Storage + Search](storage-and-search.md) |
| inspect adapter-specific facts and notebooks | [Vector Databases](vector-databases.md) |
| serve many same-process callers on one fixed hosted snapshot | [Hosted Engine Python](hosted-engine-python.md) |

## If You Want Mojo, Make Mojo Available

If your goal is to use the Mojo backend, do not stop at:

```bash
uv add kayak
```

That installs the Python package, but not the Mojo CLI. In that setup Kayak
will expose only the NumPy reference backend.

The verified path for the Mojo backend is:

1. Create or use any environment where Kayak can find a usable `mojo` CLI.
2. Install `kayak` into that same environment.
3. Use `kayak.open_text_retriever(...)` for the highest-level text workflow, or
   pass `backend=kayak.MOJO_EXACT_CPU_BACKEND` on lower-level search calls.

Any environment manager is fine as long as Kayak can find a usable `mojo` CLI.
Pixi is one easy way to make that explicit in one project. UV works too if
Mojo is already installed and discoverable through the active environment,
`PATH`, or `KAYAK_MOJO_CLI`.

Use the [installation guide](installation.md) for the exact workflow.

!!! note "Short version"
    `uv add kayak` installs the Python SDK. It does not install Mojo.
    If you want the Mojo backend, make sure Kayak can discover a usable
    `mojo` CLI in the same environment.

## What The Mojo Backend Is For

`kayak.MOJO_EXACT_CPU_BACKEND` is the exact CPU scorer for the Python SDK.

Use it when you want:

- exact local MaxSim scoring on ColBERT-style embeddings
- exact top-k search from Python
- exact reranking inside explicit search plans
- repeated queries against the same index, including batch search

Low-level search functions do not silently replace their default backend.
Kayak keeps that choice explicit on purpose.

The high-level `kayak.open_text_retriever(...)` workflow is the exception:
it prefers Mojo automatically when the active environment can actually run the
Mojo backend, and falls back to the NumPy reference backend otherwise.

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

## Highest-Level Text Workflow

If your inputs start as text, the simplest public SDK shape is now one
retriever object that composes:

- one text encoder
- one late store
- one backend default

```python
import kayak

retriever = kayak.open_text_retriever(
    encoder="colbert",
    store="kayak",
    encoder_kwargs={"model_name": "colbert-ir/colbertv2.0"},
    store_kwargs={"path": "./kayak-index"},
)

retriever.upsert_texts(doc_ids, texts, metadata=metadata_rows)
hits = retriever.search_text("install python and mojo together", k=10)
```

That is the recommended entry point when you want one Python object for text
ingest plus search.

If you are deciding which encoder to use:

- use `"colbert"` when you have a ColBERT checkpoint on Hugging Face
- use `"callable"` when you already have your own model methods

The short guide is:

- [Text Encoders](text-encoders.md)

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

- [Start Here](start-here.md) for the user-situation switchboard
- [Installation](installation.md) for the Mojo-capable install paths
- [Quickstart](quickstart.md) for a working Mojo-first script
- [Text Encoders](text-encoders.md) for ColBERT-from-HF and bring-your-own-model usage
- [Examples](examples.md) for executed notebooks and small Python scripts
- [Usage Patterns](usage-patterns.md) for which API to call for each task
- [Storage + Search](storage-and-search.md) for the vector-db-as-storage, Kayak-as-search-engine deployment shape
- [Hosted Engine Python](hosted-engine-python.md) for many same-process callers against one fixed hosted snapshot
- [Using the Mojo Backend](mojo-backend.md) for how to make Mojo your normal code path
- [Vector Database Integrations](vector-databases.md) for LanceDB, PgVector/Postgres, Qdrant, Weaviate, ChromaDB, and generic stage-1/stage-2 handoff patterns
- [Late Interaction](concepts.md) for the scoring model and explicit vector budgets
- [Search Plans](search-plans.md) for approximate stage 1 plus exact stage 2 pipelines
