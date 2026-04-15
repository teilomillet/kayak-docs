<div class="kayak-hero kayak-hero--compact" markdown>
<div class="kayak-hero__main" markdown>

<p class="kayak-eyebrow">Verified setup paths</p>

# Install Kayak in the same environment as a usable `mojo` CLI

<p class="kayak-lead">
That is the real requirement. UV, pip, and Pixi can all work. The deciding
factor is whether Kayak can actually discover a usable `mojo` command when your
Python code runs.
</p>

<div class="kayak-action-row" markdown>

[Quickstart](quickstart.md){ .md-button .md-button--primary }
[Choose your path](start-here.md){ .md-button }

</div>

</div>
<aside class="kayak-hero__aside" markdown>

<ul class="kayak-stat-list">
  <li>
    <span class="kayak-stat-label">Python-only install</span>
    <span class="kayak-stat-value"><code>numpy_reference</code> backend only</span>
  </li>
  <li>
    <span class="kayak-stat-label">Mojo-capable install</span>
    <span class="kayak-stat-value"><code>mojo_exact_cpu</code> is available and the high-level retriever prefers it automatically</span>
  </li>
  <li>
    <span class="kayak-stat-label">Best verification command</span>
    <span class="kayak-stat-value"><code>print(kayak.available_backends())</code> and <code>print(kayak.mojo_bridge_info())</code></span>
  </li>
</ul>

</aside>
</div>

## Setup Outcomes At A Glance

| Setup shape | Result |
| --- | --- |
| `uv add kayak` or `pip install kayak` with no usable `mojo` CLI | `numpy_reference` only |
| `uv add kayak` in an environment where `mojo` is already installed and discoverable | `mojo_exact_cpu` is available |
| `pixi add python=3.11 ... mojo` plus `pixi add --pypi kayak` | `mojo_exact_cpu` is available |

## Choose The Install Path That Matches Your Environment

=== "I want Mojo now"

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
    print(kayak.mojo_bridge_info())
    PY
    ```

=== "I already have Mojo"

    If a usable `mojo` CLI is already in the active environment or on `PATH`,
    you can keep using UV or pip.

    ```bash
    mojo --version
    uv add kayak
    uv run python - <<'PY'
    import kayak
    print(kayak.available_backends())
    print(kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND).availability_reason)
    print(kayak.mojo_bridge_info())
    PY
    ```

=== "I only need Python first"

    Install Kayak by itself:

    ```bash
    uv add kayak
    ```

    In that setup Kayak still works, but only the NumPy backend is available:

    ```python
    ('numpy_reference',)
    ```

## Recommended Verification

Run this after install regardless of environment manager:

```bash
python - <<'PY'
import kayak

print(kayak.available_backends())
print(kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND).availability_reason)
print(kayak.mojo_bridge_info())
print(kayak.doctor())
PY
```

Use this to verify four facts:

- the package imports correctly
- the exact backend is visible
- Kayak can explain how it found Mojo
- the public diagnostics report matches the environment you intended

## One Verified Mojo-Capable Path

Kayak currently validates the Mojo dependency in the range:

```text
>=0.26.3.0.dev2026041020,<0.27
```

One verified way to satisfy that is a Pixi workspace that already includes
Modular's `max-nightly` channel and `conda-forge`:

```bash
pixi add python=3.11 "mojo>=0.26.3.0.dev2026041020,<0.27"
pixi add --pypi kayak
```

Expected verification shape:

```python
('numpy_reference', 'mojo_exact_cpu')
True
'Kayak can invoke Mojo via: ...'
```

Pixi is not required. It is just one clear way to keep Python, Mojo, and Kayak
in the same environment.

## UV Plus A Preinstalled Mojo CLI

If the environment already owns a working `mojo` CLI, UV is also a correct
Mojo-capable setup:

```bash
mojo --version
uv add kayak
uv run python - <<'PY'
import kayak
print(kayak.available_backends())
print(kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND).availability_reason)
print(kayak.mojo_bridge_info())
PY
```

This is the right path when your environment management is already settled and
you only need Kayak to bind to that existing Mojo install.

## Optional Store Adapters

The core SDK does not require external database clients. Install only the
adapter you actually plan to use.

<div class="kayak-card-grid" markdown>

<section class="kayak-card" markdown>
### LanceDB

```bash
uv add lancedb pyarrow
```

or:

```bash
pixi add --pypi lancedb pyarrow
```
</section>

<section class="kayak-card" markdown>
### Qdrant

```bash
uv add qdrant-client
```

or:

```bash
pixi add --pypi qdrant-client
```
</section>

<section class="kayak-card" markdown>
### Weaviate

```bash
uv add weaviate-client
```

or:

```bash
pixi add --pypi weaviate-client
```
</section>

<section class="kayak-card" markdown>
### Chroma

```bash
uv add chromadb
```

or:

```bash
pixi add --pypi chromadb
```
</section>

<section class="kayak-card" markdown>
### PgVector

```bash
uv add "psycopg[binary]" pgvector
```

or:

```bash
pixi add --pypi "psycopg[binary]" pgvector
```
</section>

</div>

## What Not To Assume

If your intent is “I want the Mojo backend,” do not stop at:

```bash
uv add kayak
```

That installs the Python SDK, not Mojo itself. In that environment
`available_backends()` will normally return only:

```python
('numpy_reference',)
```

The SDK works, but the exact CPU Mojo backend is not available yet.

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
