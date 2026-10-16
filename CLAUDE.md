# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

The source of the **AI-Factory documentation site** (MkDocs + Material), published at
https://ankush610.github.io/AI-Factory-Docs/. It documents the AI-Factory platform that the install scripts in
[OOD-Install](https://github.com/Ankush610/OOD-Install) build: LDAP, Open OnDemand, MLflow, VS Code, Jupyter, Model
Hub + gateway, Keycloak, the Admin page and monitoring, on a Slurm + Kubernetes cluster.

**OOD-Install's code is the source of truth.** These pages describe what the code does. When the two disagree, the
code wins: fix the page, or record the mismatch under Known issues.

This site is the main reference for how the platform works: read it first, then the code. Plans, build logs and
the Model Hub design are private, in OOD-Install's `project/` folder (was the AI-Stack repo, now deleted): use them
for background, but never copy IPs, hostnames or secrets from them into these pages.

## Commands

```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt   # once
.venv/bin/mkdocs serve                     # preview at http://127.0.0.1:8000, reloads on save
.venv/bin/mkdocs build --strict            # what CI runs: must pass before every commit
```

`--strict` fails on broken page links. It does **not** check `#anchors`; check those too (see Checklist).
Never upgrade to MkDocs 2.0 (`requirements.txt` pins 1.6.1): Material doesn't run on it.

A push to `main` publishes the site (`.github/workflows/docs.yml`). There is no other build step.

## Repository layout

| Path | Diátaxis type | What goes there |
|---|---|---|
| `docs/index.md` | – | home page: the "pick your path" cards |
| `docs/user/` | how-to | tasks for people **using** the platform: code, train, track, deploy, call a model, FAQ |
| `docs/admin/index.md` | how-to | prerequisites checklist and the install order across all components |
| `docs/admin/install/<component>.md` | how-to | one page per OOD-Install folder (see mapping below) |
| `docs/admin/daily.md`, `troubleshooting.md`, `cluster-monitor.md` | how-to | everyday admin tasks, top problems + rerun rules, the monitoring page |
| `docs/how-it-works/NN-*.md` | explanation | what each part is, why it's built that way, diagrams |
| `docs/reference/` | reference | `site.conf`, scripts, ports, files/accounts/secrets, admin actions, monitoring metrics/alerts |
| `docs/known-issues/` | reference | bugs, security gaps, doc mismatches found in OOD-Install's code |
| `docs/glossary.md` | reference | one alphabetical table of terms |
| `mkdocs.yml` | – | theme, Markdown extensions, and the **`nav:`** (every page must be listed here) |

**Keep the types apart.** A how-to page gives steps; it links to the explanation instead of explaining. A
how-it-works page explains; it links to the install page instead of listing commands. A reference page lists facts
in tables; it doesn't teach. If a page starts mixing, move the text to the right page and link it.

## Which pages a change touches

When OOD-Install changes, start from the folder that changed:

| OOD-Install folder | Install page | How it works | Reference | Known issues |
|---|---|---|---|---|
| `site.conf` | the components that use the variable | – | `reference/site-conf.md` | – |
| `0-registry` | `admin/install/registry.md` | `06-containers-and-registry.md` | ports, files | `registry.md` |
| `1-ldap` | `admin/install/ldap.md` | `03-users-and-login.md` | ports, files | `ldap.md` |
| `2-ood` | `admin/install/portal.md` | `04-web-portal-ood.md` | ports, files | `portal.md` |
| `3-mlflow` | `admin/install/mlflow.md` | `08-mlflow.md` | ports, files | `mlflow.md` |
| `4-vscode` | `admin/install/vscode.md` | `07-vscode-and-jupyter.md`, `05-slurm-and-kubernetes.md` | – | `vscode.md` |
| `5-model-hub` | `admin/install/model-hub.md` | `09-model-hub.md`, `10-gateway-and-api-keys.md` | ports, files | `model-hub.md` |
| `6-keycloak` | `admin/install/keycloak.md` | `03-users-and-login.md` | ports, files | `keycloak.md` |
| `7-jupyter` | `admin/install/jupyter.md` | `07-vscode-and-jupyter.md` | – | `jupyter.md` |
| `8-admin` | `admin/install/admin-page.md` | `11-security-and-isolation.md` | `admin-actions.md` | `admin-page.md` |
| `9-monitoring` | `admin/install/monitoring.md` | `12-monitoring.md` | `monitoring.md` | `monitoring.md` |

Also check, for every change:

- **A new or renamed script:** `reference/scripts.md`, the install table in `admin/index.md`.
- **A new port or firewall rule:** `reference/ports-and-firewall.md`, `how-it-works/01-cluster-architecture.md`.
- **A new path, account, UID, secret or namespace:** `reference/files-and-accounts.md`.
- **Something users see** (a button, a page, a message, a limit): the matching `docs/user/` page and `user/faq.md`.
- **A new everyday task or "rerun X after Y" rule:** `admin/daily.md`, `admin/troubleshooting.md`.
- **A security-relevant change:** `how-it-works/11-security-and-isolation.md`, `known-issues/security.md`.
- **A new term:** `glossary.md`.
- **A new component (new numbered folder):** a new install page, how-it-works page and known-issues page; add them
  to `nav:`, the mapping table above, `admin/index.md`, `how-it-works/index.md`, `known-issues/index.md`.

Find every mention before you edit: `grep -rn "<script name|variable|old value>" docs/`.

## How to update the docs after the tool changes

1. **Read the change in OOD-Install**, not just its commit message: `git log`/`git diff` of the scripts, their
   `site.conf` variables, the folder README and its Symptom | Cause | Fix table. The OOD-Install checkout may be at
   `../OOD-Install`; otherwise clone `https://github.com/Ankush610/OOD-Install`.
2. **Find every affected page** with the mapping table and `grep`.
3. **Update facts in place.** Change the value, step, message or row; don't append "update:" notes. Keep each
   script's `== N.` step names and quote error messages exactly, so readers can match what they see.
4. **Known issues:** delete entries the change fixed (and their row in that page's summary table and, if listed,
   in `known-issues/index.md` "Fix these first"). Add new ones you find.
5. **Build and check** (see Checklist), then commit.

If you can't confirm something from the code, say so: write **(inferred)** in reference and known-issues pages, or
leave it out of how-to pages. Never invent a flag, port, path, value or behaviour.

## Writing style

Based on the Google developer documentation style guide:

- Second person ("you"), present tense, active voice. Short sentences, one idea each. Plain words; explain a
  technical term the first time, or link the glossary.
- Lead with the purpose: the first lines of a page say what it is and who it is for.
- Bold for UI elements and key words: **Restart Web Server**, **Interactive Apps → Model Hub**. Arrows `→` for menu
  paths. Code font for commands, files, variables, values: `site.conf`, `OOD_AUTH=keycloak`.
- Commands in fenced blocks with a language tag (`bash`, `text`, `yaml`), with a comment saying **where** they run
  (`# on master`, `# on the login node`) when it matters. Say when a script needs `sudo` and when it must **not**.
- Tables for anything with more than two parallel facts. Bullets over long paragraphs.
- Mermaid diagrams (```` ```mermaid ````) for real flows: requests, logins, data. Not for decoration.
- Admonitions sparingly: `!!! warning` for things that break a cluster, `!!! note` / `!!! tip` for side facts.
- Use "the login node", "master", "the portal" in prose; use the `site.conf` names (`LOGIN_HOST`, `MASTER_HOST`)
  in tables and commands.

## Public repository: never commit

This repo is **public**. Never put in it:

- passwords, tokens, keys, hashes of real secrets;
- internal IPs, the jump host, real hostnames other than the generic `master`, `cn01`, `cn02`;
- customer names, real usernames from the cluster.

Use placeholders: `<portal>`, `<gateway>`, `<master-ip>`, `<gateway-ip>`, `<jump-host>`, `<login-node>`, `<user>`.
`site.conf` example values are fine **except** IP addresses. Before committing:

```bash
grep -rnE "([0-9]{1,3}\.){3}[0-9]{1,3}" docs/ | grep -vE "127\.0\.0\.1|0\.0\.0\.0"    # should print nothing
```

## Page templates

### Install page (`docs/admin/install/<component>.md`)

```markdown
# N · Component name

> One or two sentences: what it gives the platform.

**Folder:** [`OOD-Install/N-folder`](https://github.com/Ankush610/OOD-Install/tree/main/N-folder) ·
**Explained in:** [How it works page](../../how-it-works/NN-x.md) ·
**Known issues:** [Component](../../known-issues/x.md)

## What it sets up        (bullets)
## Before you start       (table: script | runs on | root; then prerequisites)
## Settings used          (table: variable | example | meaning; link reference/site-conf.md)
## Install                (### 1. Step (host): command block, step table, **Expect:** …)
## Everyday tasks         (table: task | how)
## Troubleshooting        (table: Symptom | Cause | Fix — every row from the folder README + script error messages)
```

### Known issue entry (`docs/known-issues/<component>.md`)

Each page starts with a summary table `| Issue | Type | Confidence |` linking to its entries. Each entry:

```markdown
## Short title of the problem

- **Type:** Bug | Security | Docs | Gap | Question · **Confidence:** Confirmed | Inferred · **Where:** `path/file.sh:12-20`
- **What happens:** the effect in plain words, and when it matters.
- **Workaround:** what to do until it is fixed (if any).
- **Fix:** the change that would solve it (if clear).
```

**Confirmed** = read directly in the code. **Inferred** = follows from the code, not tested on a cluster. `Where`
paths are relative to the OOD-Install root. Small items go in a "Smaller items" bullet list at the end.

### How-it-works page (`docs/how-it-works/NN-*.md`)

`## What it is` · `## Why we need it` · `## How it works` (diagram) · `## What the user sees` ·
`## What the admin does` (link the install page) · `## Limits and known issues` (link the known-issues page).

## Links and anchors

- Link pages by **relative path to the `.md` file**: `[Portal](../admin/install/portal.md)`. Never link to the
  built site's URLs.
- Heading anchors are made by MkDocs' `toc` slugify: lower case, spaces → `-`, punctuation and backticks dropped.
  `## Keycloak with an empty \`MLFLOW_PREFIX\` breaks Open MLflow` →
  `#keycloak-with-an-empty-mlflow_prefix-breaks-open-mlflow`. **Renaming a heading breaks links to it**:
  `grep -rn "#old-anchor" docs/` and fix them.
- Link OOD-Install files and folders with full GitHub URLs (`https://github.com/Ankush610/OOD-Install/tree/main/…`).
- `RUNBOOK_URL` in OOD-Install points at `reference/monitoring/` on this site: the Cluster monitor links
  `#<alertname in lower case>` and `#alerts-what-to-do`. Keep those `<a id>` anchors in `reference/monitoring.md`;
  a new alert in `rules.yml`/`node-alerts.yml` needs a row with its anchor there.

## Checklist before every commit

- [ ] Facts match OOD-Install's code (values, step names, error messages, hosts, root or not).
- [ ] Every new page is in `mkdocs.yml` `nav:` and linked from its section's index.
- [ ] Fixed known issues are deleted; new ones added with Type, Confidence and Where.
- [ ] `mkdocs build --strict` passes.
- [ ] Anchors resolve. To check, build with anchor validation (should print no `WARNING`):

    ```bash
    cp mkdocs.yml /tmp/mk.yml && printf '\ndocs_dir: %s/docs\nvalidation:\n  anchors: warn\n' "$PWD" >> /tmp/mk.yml
    .venv/bin/mkdocs build -f /tmp/mk.yml -d /tmp/site 2>&1 | grep WARNING
    ```

- [ ] No IPs, secrets or real names (the `grep` above).
- [ ] Commit message says which OOD-Install change the docs follow (folder, and commit if known).
