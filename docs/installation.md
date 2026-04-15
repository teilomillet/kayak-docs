# Installation

## The Short Version

If you want the Mojo backend, install `kayak` into a Pixi environment that
already has Mojo.

If you install only the Python package with `uv add kayak` or `pip install
kayak`, Kayak will still work, but it will default to the NumPy reference
backend because no Mojo CLI is available.

Kayak wheels bundle the Mojo backend they were built with. If your first
`mojo_exact_cpu` call reports that the bundled backend and the active Mojo
compiler do not match, upgrade `kayak` and `mojo` together so both come from
compatible releases.

That distinction is verified in the code:

- `numpy_reference` is always available
- `mojo_exact_cpu` appears only when Kayak can find a usable `mojo` command
- Kayak never auto-switches from NumPy to Mojo just because Mojo is present

## Recommended: Pixi Plus Mojo Plus Kayak

Kayak currently validates the Mojo dependency in the range:

```text
>=0.26.3.0.dev2026041020,<0.27
```

Inside a Pixi workspace that already includes Modular's `max-nightly` channel
and `conda-forge`, install the Python runtime, Mojo, and Kayak together:

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

This is the setup to recommend if you want your normal development environment
to be Mojo-capable.

## If You Already Have Mojo Installed

If you already have a working `mojo` CLI in the same Python environment, or on
`PATH`, installing the Python package is enough:

```bash
pip install kayak
mojo --version
python - <<'PY'
import kayak
print(kayak.available_backends())
PY
```

That is a valid setup. It is just not the best starting point when your goal is
"use Kayak with Mojo by default in my app" because it leaves the environment
contract implicit.

## Optional Store Adapters

The core SDK does not require external database clients.

Install only the extra adapter you actually plan to use.

For the public LanceDB store adapter:

```bash
pixi add --pypi lancedb pyarrow
```

or, outside Pixi:

```bash
uv add lancedb pyarrow
```

Then `kayak.open_store("lancedb", ...)` becomes available in that environment.

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

Installing with Pixi plus Mojo makes the backend available. It does not make the
SDK silently switch defaults.

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
