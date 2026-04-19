# Install Kayak

Install the Python package first.

```bash
uv add kayak
```

Or:

```bash
pip install kayak
```

## Recommended Verification

Run this after install:

```bash
python - <<'PY'
import kayak

print(kayak.available_backends())
print(kayak.doctor())
PY
```

Use this to verify:

- the package imports correctly
- the available backend set matches the environment you expected
- the public diagnostics report matches the environment you intended

## Optional Exact Backend

The Python SDK always supports the reference backend:

- `kayak.NUMPY_REFERENCE_BACKEND`

Some environments may also expose the optional exact CPU backend:

- `kayak.MOJO_EXACT_CPU_BACKEND`

Check what the current environment can actually use:

```bash
python - <<'PY'
import kayak

print(kayak.available_backends())
print(kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND))
PY
```

## Optional Store Adapters

The core SDK does not require external database clients. Install only the
adapter you plan to use.

| Adapter | Install |
| --- | --- |
| LanceDB | `uv add lancedb pyarrow` or `pip install lancedb pyarrow` |
| Qdrant | `uv add qdrant-client` or `pip install qdrant-client` |
| Weaviate | `uv add weaviate-client` or `pip install weaviate-client` |
| Chroma | `uv add chromadb` or `pip install chromadb` |
| PgVector | `uv add "psycopg[binary]" pgvector` or `pip install "psycopg[binary]" pgvector` |

## Useful Next Pages

| If you want to... | Open... |
| --- | --- |
| run one exact search from vectors | [Quickstart](quickstart.md) |
| start from raw text | [Usage Patterns](usage-patterns.md) |
| keep an existing database | [Storage + Search](storage-and-search.md) |
| inspect the public surface | [Python API](api.md) |
