# AI-Factory-Docs

Source of the AI-Factory documentation site: https://ankush610.github.io/AI-Factory-Docs/

Pages are Markdown in `docs/`; the menu is `nav:` in `mkdocs.yml`. A push to `main` rebuilds and publishes the
site (`.github/workflows/docs.yml`).

Preview locally:

```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
.venv/bin/mkdocs serve        # http://127.0.0.1:8000, reloads on save
```

`mkdocs build --strict` (what CI runs) fails on broken links.

This repo is **public**: no passwords, internal IPs or customer names. Write `<portal>`, `<gateway>` instead.
Install scripts: [OOD-Install](https://github.com/Ankush610/OOD-Install). Images: [OOD-Container](https://github.com/Ankush610/OOD-Container).
