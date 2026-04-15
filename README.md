# kayak-docs

Documentation for [kayak](https://pypi.org/project/kayak) — late-interaction retrieval for Python.

Live at [docs.withkayak.com](https://docs.withkayak.com).

## Local development

```bash
pip install -r requirements.txt
mkdocs serve
```

Open [http://localhost:8000](http://localhost:8000).

The requirements file intentionally keeps the site on MkDocs 1.x and Material
9.x for now. This avoids drifting into the MkDocs 2.x path before the upstream
theme story is settled.

## Deploy

Documentation deploys automatically on push to `main` via GitHub Actions.
