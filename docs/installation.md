# Installation

## The Short Version

Any Python environment manager works.

If you want the Mojo backend, Kayak needs to be able to find a usable `mojo`
CLI. That is the actual requirement.

If you install only the Python package with `uv add kayak` or `pip install
kayak`, and no usable Mojo CLI is visible, Kayak will still work but it will
default to the NumPy reference backend.

Verified setup shapes:

| Setup | Result |
| --- | --- |
| `uv add kayak` or `pip install kayak` with no usable `mojo` CLI | `numpy_reference` only |
| `uv add kayak` in an environment where `mojo` is already installed and discoverable | `mojo_exact_cpu` is available |
| `pixi add python=3.11 ... mojo` plus `pixi add --pypi kayak` | `mojo_exact_cpu` is available |

Kayak wheels bundle the Mojo backend they were built with. If your first
`mojo_exact_cpu` call reports that the bundled backend and the active Mojo
compiler do not match, upgrade `kayak` and `mojo` together so both come from
compatible releases.

That distinction is verified in the code:

- `numpy_reference` is always available
- `mojo_exact_cpu` appears only when Kayak can find a usable `mojo` command
- low-level operations like `search(...)` and `maxsim(...)` never auto-switch from NumPy to Mojo
- `open_text_retriever(...)` prefers Mojo automatically when the backend is actually available

## Choose Your Install Path

=== "I Want Mojo Now"

    Use one environment where Python, Mojo, and Kayak live together.

    ```bash
    pixi add python=3.11 "mojo>=0.26.3.0.dev2026041020,<0.27"
    pixi add --pypi kayak
    ```

    Then verify:

    ```bash
    pixi run python - <<'PY'
    import kayak
    print(kayak.available_backends())
    print(kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND).availability_reason)
    PY
    ```

    Use this when you want the shortest verified path to `mojo_exact_cpu`.

=== "I Already Have Mojo"

    If a usable `mojo` CLI is already in the active environment or on `PATH`,
    you can keep using UV or pip.

    ```bash
    mojo --version
    uv add kayak
    uv run python - <<'PY'
    import kayak
    print(kayak.available_backends())
    print(kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND).availability_reason)
    PY
    ```

    Use this when your environment already owns Mojo and you just need Kayak to
    see it.

=== "I Only Need Python First"

    Install Kayak by itself:

    ```bash
    uv add kayak
    ```

    In that setup Kayak works, but only the NumPy backend is available:

    ```python
    ('numpy_reference',)
    ```

    Use this when you want to start with the Python SDK now and add Mojo later.

## One Verified Way: One Environment With Mojo Plus Kayak

Kayak currently validates the Mojo dependency in the range:

```text
>=0.26.3.0.dev2026041020,<0.27
```

One verified way to do that is a Pixi workspace that already includes
Modular's `max-nightly` channel and `conda-forge`. Install the Python runtime,
Mojo, and Kayak together:

```bash
pixi add python=3.11 "mojo>=0.26.3.0.dev2026041020,<0.27"
pixi add --pypi kayak
```

Then verify that Kayak can see the backend:

```bash
pixi run python - <<'PY'
import kayak

print(kayak.available_backends())
info = kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND)
print(info.available)
print(info.availability_reason)
PY
```

Expected shape:

```python
('numpy_reference', 'mojo_exact_cpu')
True
'Kayak can invoke Mojo via: ...'
```

This is an easy Mojo-capable setup, but it is not the only valid setup.

## UV Plus A Preinstalled Mojo CLI

If you already have a working `mojo` CLI in the same Python environment, or on
`PATH`, UV is enough too:

```bash
mojo --version
uv add kayak
uv run python - <<'PY'
import kayak
print(kayak.available_backends())
print(kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND).availability_reason)
PY
```

That is a valid Mojo-capable setup.

UV is correct when the environment already owns Mojo. Pixi is just another way
to make that environment explicit in one place.

## Optional Store Adapters

The core SDK does not require external database clients.

Install only the extra adapter you actually plan to use.

For the public LanceDB store adapter:

```bash
uv add lancedb pyarrow
```

or:

```bash
pixi add --pypi lancedb pyarrow
```

Then `kayak.open_store("lancedb", ...)` becomes available in that environment.

For the public Qdrant store adapter:

```bash
uv add qdrant-client
```

or:

```bash
pixi add --pypi qdrant-client
```

Then `kayak.open_store("qdrant", ...)` becomes available.

For the public Weaviate store adapter:

```bash
uv add weaviate-client
```

or:

```bash
pixi add --pypi weaviate-client
```

Then `kayak.open_store("weaviate", ...)` becomes available.

For the public Chroma store adapter:

```bash
uv add chromadb
```

or:

```bash
pixi add --pypi chromadb
```

Then `kayak.open_store("chromadb", ...)` becomes available.

For the public pgvector store adapter:

```bash
uv add "psycopg[binary]" pgvector
```

or:

```bash
pixi add --pypi "psycopg[binary]" pgvector
```

Then `kayak.open_store("pgvector", ...)` becomes available.

## What Not To Do

If your intent is "I want the Mojo backend," do not document or recommend only:

```bash
uv add kayak
```

That installs the SDK, not Mojo itself. In that environment `available_backends()`
will normally return only:

```python
('numpy_reference',)
```

The package works, but the exact CPU Mojo backend is not installed yet.

## How Kayak Finds Mojo

Kayak discovers Mojo in this order:

1. `KAYAK_MOJO_CLI`
2. a usable `mojo` binary in the active Python environment
3. `mojo` on `PATH`
4. `pixi run mojo`

If you have multiple Mojo installs, set `KAYAK_MOJO_CLI` explicitly:

```bash
export KAYAK_MOJO_CLI=/full/path/to/mojo
```

`KAYAK_MOJO_CLI` can also be a full command prefix when you need a wrapper:

```bash
export KAYAK_MOJO_CLI="bash /full/path/to/run_mojo_with_wrapper.sh"
```

That removes ambiguity and is the right choice for CI or shared dev machines.

## The Backend Still Stays Explicit In Code

Any setup that makes a usable `mojo` CLI visible to Kayak makes the backend
available. The low-level SDK still keeps backend choice explicit.

The one high-level exception is `open_text_retriever(...)`, which prefers Mojo
automatically when the active environment can run it.

If you want your application code to behave as "Mojo by default," define the
backend once and reuse it:

```python
import kayak

BACKEND = kayak.MOJO_EXACT_CPU_BACKEND

hits = kayak.search(query, index, k=10, backend=BACKEND)
scores = kayak.maxsim(query, index, backend=BACKEND)
result = kayak.search_with_plan(query, index, plan, backend=BACKEND)
```

That pattern is explicit, easy to grep, and easy to override in tests.

## First Use Behavior

The first call that uses the Mojo backend can take longer than steady-state
calls. Kayak may need to:

- detect the Mojo CLI
- build a `kayak.mojopkg`
- build a Python extension module for the backend
- prepare reusable index state for exact search

Do not treat the first Mojo call as representative of steady-state latency.

## Backend Inspection API

Use these methods to verify the runtime contract instead of guessing:

```python
import kayak

print(kayak.available_backends())

info = kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND)
print(info.name)
print(info.available)
print(info.requires_mojo)
print(info.query_layouts)
print(info.index_layouts)
print(info.availability_reason)
```

`available_backends()` returns a tuple of backend names, not a list.

## Recommended Reading After Install

- [Quickstart](quickstart.md) for the shortest working Mojo-first script
- [Using the Mojo Backend](mojo-backend.md) for how to make Mojo your normal application path
