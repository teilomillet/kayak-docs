# Text Encoders

This page is the shortest answer to:

- what encoder to use
- how to pass a Hugging Face model
- how to bring your own model into Kayak

This page starts after document parsing. If your source data is PDFs, scans,
tables, or mixed-layout files, use a parser or extractor first and pass Kayak
plain text or token-level vectors.

Keep this simple:

1. if you already have a ColBERT checkpoint on Hugging Face, use the built-in
   ColBERT encoder
2. if you already have your own model that emits token-level vectors, use the
   callable encoder
3. if you want one object for plain-text ingest plus search, pass that encoder into
   `kayak.open_text_retriever(...)`

## Choose The Encoder Path

=== "I Have A Hugging Face ColBERT Checkpoint"

    Use:

    - `kayak.open_encoder("colbert", model_name="...")`

    Choose this when your model is already a ColBERT checkpoint and you want the
    shortest supported setup.

=== "I Have My Own Model Methods"

    Use:

    - `kayak.open_encoder("callable", query_encoder=..., document_encoder=...)`

    Choose this when you already control the model wrapper and only need Kayak
    to consume the token-level vectors it emits.

=== "I Want One Search Object"

    Use:

    - `kayak.open_text_retriever(...)`

    Choose this when your application starts from plain text and you want one SDK
    object that owns encoder plus store wiring.

## The Two Main Paths

### 1. ColBERT From A Hugging Face Repo Id

Use this when your model is already a ColBERT checkpoint.

```python
import kayak

encoder = kayak.open_encoder(
    "colbert",
    model_name="colbert-ir/colbertv2.0",
)
```

`model_name` is the Hugging Face repo id.

You can use that encoder directly:

```python
query = encoder.encode_query("install python and mojo together")
documents = encoder.encode_documents(doc_ids, texts)
index = documents.pack()
hits = kayak.search(
    query,
    index,
    k=10,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

Or use it in the highest-level workflow:

```python
retriever = kayak.open_text_retriever(
    encoder="colbert",
    store="kayak",
    encoder_kwargs={"model_name": "colbert-ir/colbertv2.0"},
    store_kwargs={"path": "./kayak-index"},
)

retriever.upsert_texts(doc_ids, texts, metadata=metadata_rows)
hits = retriever.search_text("install python and mojo together", k=10)
```

Runnable example:

- `python/examples/colbert_hf_encoder.py`

## 2. Bring Your Own Model

Use this when you already have a model or wrapper that can turn one text string
into one token-level vector matrix.

The only requirement is the shape:

- one 2D matrix per text
- rows are token vectors
- columns are the shared vector dimension

Then pass those methods to the callable encoder:

```python
import kayak

encoder = kayak.open_encoder(
    "callable",
    query_encoder=my_model.encode_query_tokens,
    document_encoder=my_model.encode_document_tokens,
)
```

That encoder can be used directly:

```python
query = encoder.encode_query(query_text)
documents = encoder.encode_documents(doc_ids, texts)
index = documents.pack()
hits = kayak.search(
    query,
    index,
    k=10,
    backend=kayak.MOJO_EXACT_CPU_BACKEND,
)
```

Or in the higher-level retriever:

```python
retriever = kayak.open_text_retriever(
    encoder=encoder,
    store="memory",
    backend=kayak.NUMPY_REFERENCE_BACKEND,
)
```

Runnable example:

- `python/examples/byo_model_encoder.py`

## What Your Model Must Return

Your model does not need to know about `LateQuery` or `LateDocuments`.
Kayak wraps that for you.

Your query/document methods only need to return something array-like that looks
like:

```python
[
    [0.1, 0.0, 0.4, ...],
    [0.0, 0.3, 0.2, ...],
    ...
]
```

The rules are:

- one query returns one 2D token matrix
- one document returns one 2D token matrix
- vector count can change from one text to another
- vector dimension must stay the same inside one retrieval setup

Current public encoder behavior is intentionally simple:

- `encode_query(...)` is one text at a time
- `encode_documents(...)` is one document text at a time behind the SDK surface

If your model already has an efficient batch API, keep using that inside your
own wrapper and pass the single-text methods that delegate to it where
appropriate.

## The Simplest Full Workflow

If you want the shortest late-interaction setup and your inputs start as plain
text:

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

If you already have your own model:

```python
encoder = kayak.open_encoder(
    "callable",
    query_encoder=my_model.encode_query_tokens,
    document_encoder=my_model.encode_document_tokens,
)

retriever = kayak.open_text_retriever(
    encoder=encoder,
    store="kayak",
    store_kwargs={"path": "./kayak-index"},
)
```

## What To Ignore At First

You do not need to think about:

- vector database adapters
- stage-1 candidate generation
- reranking plans
- verifier operators

Start with:

- one encoder
- one store
- one retriever
- one query

Then add the rest only if you actually need it.

## Read Next

- [Usage Patterns](usage-patterns.md)
- [Quickstart](quickstart.md)
- [Storage + Search](storage-and-search.md)
