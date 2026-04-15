# Installation

## Requirements

- Python 3.11 or later
- NumPy

## Install

```bash
pip install kayak
```

With UV:

```bash
uv add kayak
```

With Pixi:

```bash
pixi add --pypi kayak
```

---

## Backends

Kayak exposes two backends. You choose explicitly — nothing switches automatically.

### `numpy_reference`

The default. Pure NumPy, always available, no extra dependencies.

```python
import kayak

hits = kayak.search(query, index, k=10, backend=kayak.NUMPY_REFERENCE_BACKEND)
```

Use this for development, testing, and any environment where Mojo isn't available.

### `mojo_exact_cpu`

Mojo-backed exact CPU scoring. Faster than the NumPy path for larger workloads.

Requires a Mojo installation in your environment.

```python
hits = kayak.search(query, index, k=10, backend=kayak.MOJO_EXACT_CPU_BACKEND)
```

Kayak discovers Mojo in this order:

1. `KAYAK_MOJO_CLI` environment variable
2. `mojo` in the active Python environment
3. `mojo` on `PATH`
4. `pixi run mojo`

---

## Check available backends

```python
import kayak

print(kayak.available_backends())
# ['numpy_reference', 'mojo_exact_cpu']  — if Mojo is installed
# ['numpy_reference']                    — otherwise

info = kayak.backend_info(kayak.MOJO_EXACT_CPU_BACKEND)
print(info)
```

---

## Install with Mojo (Pixi)

```bash
pixi init .
pixi add python=3.11 mojo
pixi add --pypi kayak
```
