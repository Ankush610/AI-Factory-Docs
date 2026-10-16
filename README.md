# AI-Factory-Docs

Source of the AI-Factory documentation site: https://ankush610.github.io/AI-Factory-Docs/

Pages are Markdown in `docs/`; the menu is `nav:` in `mkdocs.yml`.

| Folder | Kind of page |
|---|---|
| `docs/user/` | how-to for people using the platform |
| `docs/admin/` | how-to for admins; `admin/install/` has one install page per OOD-Install folder |
| `docs/how-it-works/` | explanation: what each part is and why it is built that way |
| `docs/reference/` | facts to look up: `site.conf`, scripts, ports, paths, admin actions, alerts |
| `docs/known-issues/` | bugs and gaps found in OOD-Install's code. Delete an entry when it is fixed there |

Each page sticks to one kind (tutorial/how-to, explanation or reference). Facts come from the OOD-Install code;
anything not checked on a running cluster is marked **inferred**. A push to `main` rebuilds and publishes the
site (`.github/workflows/docs.yml`).

Preview locally:

```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
.venv/bin/mkdocs serve        # http://127.0.0.1:8000, reloads on save
```

`mkdocs build --strict` (what CI runs) fails on broken links.

This repo is **public**: no passwords, internal IPs or customer names. Write `<portal>`, `<gateway>` instead.
Install scripts: [OOD-Install](https://github.com/Ankush610/OOD-Install). Images: [OOD-Container](https://github.com/Ankush610/OOD-Container).
