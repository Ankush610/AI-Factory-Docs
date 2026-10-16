# 7 · Jupyter in the browser

> Every user gets their own JupyterLab, started as a Slurm job that runs as them, from a portal page it shares
> with VS Code.

**Folder:** [`OOD-Install/7-jupyter`](https://github.com/Ankush610/OOD-Install/tree/main/7-jupyter) ·
**Explained in:** [VS Code and Jupyter](../../how-it-works/07-vscode-and-jupyter.md) ·
**Known issues:** [Jupyter](../../known-issues/jupyter.md)

## What it sets up

- **One JupyterLab install** on the shared `/home` (`JUPYTER_ROOT/<lab>-py<python>` and a `current` link) with its
  **own Python 3.12** (python-build-standalone) and a venv on top. JupyterLab needs Python ≥ 3.10; the nodes have 3.9.
- **The portal page** **Interactive Apps → Jupyter**. It is VS Code's page (`4-vscode/…/passenger_wsgi.py`),
  copied as-is; `site.json` `"app": "jupyter"` picks the Jupyter texts.
- **Sessions** on the `viewer` partition ("Notebook only": 1 shared CPU, `JUPYTER_MEM`) or on a GPU node
  ("GPU node: n × <type>").
- **Kernels built at session start**, one per training image `CONTAINERS_ROOT/training/*/*.sif`, run with
  `apptainer exec --nv`. A new image needs no install step.

## Before you start

| Script | Runs on | Root |
|---|---|---|
| `1-install-jupyter.sh` | master | yes |
| `2-install-ood-app.sh` | login node | yes |

- **Done first:** [Portal](portal.md) and [VS Code](vscode.md) (at least the `viewer` partition). The installer reads
  the page and `gpu-detect.sh` from `4-vscode/` in the repo.
- **For the page's look:** the [Model Hub](model-hub.md) portal app (same as VS Code).
- **Optional:** Apptainer at `APPTAINER` and training images under `CONTAINERS_ROOT`. Without them, only the plain
  Python kernel exists. OOD-Install doesn't install either.
- **Internet on master:** the python-build-standalone tarball from GitHub and the PyPI packages in `requirements.txt`.

## Settings used

| Variable | Example | Meaning |
|---|---|---|
| `JUPYTER_ROOT` | `/home/apps/jupyter` | the shared install |
| `JUPYTER_PYTHON` | `3.12.15+20261003` | python-build-standalone `<version>+<release date>` |
| `JUPYTER_MEM` | `4G` | memory per "Notebook only" session |
| `JUPYTER_IDLE_SECONDS` | `3600` | idle kernels stop after this; then the server stops |
| `OOD_SERVERNAME` | `<portal>` | becomes Jupyter's `allow_origin`. Must be the exact address users type |
| `CONTAINERS_ROOT`, `APPTAINER` | `/home/apps/containers`, `/home/apps/apptainer/bin/apptainer` | training images and the binary that runs them |
| `VIEWER_PARTITION`, `GPU_PARTITION` | `viewer`, `auto` | where sessions run |

Fixed in the code: session hours 1–12 (default 4), kernel check every 300 s, session logs kept 7 days, 120 s
start-up wait.

## Install

### 1. Python and JupyterLab (master)

```bash
sudo bash 7-jupyter/1-install-jupyter.sh
```

| Step | What it does |
|---|---|
| `== 1. Python` | downloads python-build-standalone (resuming, up to 5 tries, `gzip -t`), unless already there |
| `== 2. JupyterLab` | creates the venv, `pip install -r requirements.txt` (10 retries); removes old container kernels from the venv; points `current` at the new folder |
| `== 3. Check` | `jupyter lab --version` and `kernelspec list` on master, then the version on each compute node through `srun` |

**Expect:** a version such as `4.6.4` for master and each compute node, then `Done. Next: sudo bash 2-install-ood-app.sh`.

### 2. The portal page (login node)

```bash
sudo bash 7-jupyter/2-install-ood-app.sh
```

| Step | What it does |
|---|---|
| `== 1. Checks` | stops if the portal, JupyterLab or the `viewer` partition is missing; warns if Apptainer or the training images are missing; lists each image as `kernel: <path>` |
| `== 2. GPUs` | detects the GPU partition and sizes from `sinfo` |
| `== 3. Copy` | recreates `/var/www/ood/apps/sys/jupyter` from 4-vscode's page and this folder's `manifest.yml` and `job.sh`; writes `site.json` |
| `== 4. Check` | no unfilled `${…}` in `job.sh`; syntax checks; prints `site.json`; warns if Model Hub's app is missing |

**Expect:** `job.sh: OK`, `passenger_wsgi.py: OK`, `site.json` with `"app": "jupyter"`, then
`Done. In OOD: Restart Web Server, then Interactive Apps -> Jupyter.`

### 3. Check as a user

Restart Web Server. Open **Interactive Apps → Jupyter**, launch "Notebook only", wait for **Ready**, click
**Connect**. JupyterLab opens in a new tab with the `(container)` kernels in the launcher.

## Everyday tasks

| Task | How |
|---|---|
| Add a training image as a kernel | copy it to `CONTAINERS_ROOT/training/<family>/<name>.sif`. It must contain `ipykernel`. The next session offers it |
| Upgrade JupyterLab | in a scratch venv on the standalone Python, `pip install jupyterlab==<new> ipykernel` and `pip freeze` into `requirements.txt` (keep its two comment lines); then rerun `1-install-jupyter.sh`. Running sessions keep the old folder |
| Upgrade Python | set `JUPYTER_PYTHON` to another `<version>+<date>`, rerun `1-install-jupyter.sh` |
| Clean up old versions | delete old `JUPYTER_ROOT/<lab>-py<python>` folders once no session uses them (`squeue -n jupyter`) |
| GPU nodes, `OOD_SERVERNAME`, `JUPYTER_MEM` or `JUPYTER_IDLE_SECONDS` changed | rerun `2-install-ood-app.sh`, Restart Web Server. New sessions only |
| Changed the shared page | rerun **both** `4-vscode/3-install-ood-app.sh` and `7-jupyter/2-install-ood-app.sh` |
| See and stop sessions | `squeue -n jupyter`, `squeue -p viewer`, `scancel <job>`; or Admin page → Running → Stop |
| Read a session's log | `~<user>/.jupyter-sessions/<job>.log` (kept 7 days) |

Example scratch venv for the upgrade (any path works):

```bash
/home/apps/jupyter/current/python/bin/python3 -m venv /tmp/jl-new
/tmp/jl-new/bin/pip install jupyterlab==<new> ipykernel
/tmp/jl-new/bin/pip freeze
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Download failed` / pip `Read timed out` | the internet dropped | rerun (download resumes, pip retries) |
| `<node>: (srun failed, check by hand)` | `srun` to that node failed | run `JUPYTER_ROOT/current/bin/jupyter lab --version` there; check Slurm and the `/home` mount |
| `Run this on <LOGIN_HOST>` | wrong host | run on the login node |
| `No /var/www/ood/apps/sys` / `No JupyterLab` / `No partition viewer` | an earlier step is missing (or `/home` isn't mounted) | run the missing step |
| `WARNING: no <APPTAINER>` / `no …/training/*/*.sif` | Apptainer or images missing | install Apptainer there / copy the images in; or fix the paths |
| `partition '<x>' has no GPUs` | `GPU_PARTITION` names a partition without GPUs | `auto`, a listed partition, or `""` |
| `unfilled values above` | a `${…}` in `job.sh` wasn't filled | check that variable in `site.conf` |
| Session stays **Waiting** | `viewer` is out of memory, or no GPU is free | `squeue -p viewer`; lower `JUPYTER_MEM`/`VSCODE_MEM`, or wait |
| Session starts, then disappears; log: `jupyter did not start` | wrong `JUPYTER_ROOT`, or Jupyter crashed | `JUPYTER_ROOT/current/bin/jupyter lab --version` on that node; read the log |
| Notebook opens but the kernel stays "Connecting…" | the browser's address isn't `https://<OOD_SERVERNAME>` | open the portal by that name, or fix `OOD_SERVERNAME` and rerun |
| **Connect** shows Jupyter's login page | the token didn't match | relaunch |
| A container kernel dies at once | the image has no `ipykernel`, or was moved | add `ipykernel` to its recipe and rebuild; start a new session |
| A new image doesn't show | the session started before the copy, or the path is wrong | start a new session; use `training/<family>/<name>.sif` |
| `torch.cuda.is_available()` is False | a "Notebook only" session, or a non-container kernel | launch a GPU session, pick a `(container)` kernel |
| `sbatch` from Jupyter's terminal fails | the session's own Slurm variables leaked | `job.sh` clears them; rerun the installer, start a new session |
| GPU choices wrong after GPU changes | fixed at install | rerun `2-install-ood-app.sh`, Restart Web Server |
| Page: `Could not ask Slurm for your sessions` | `SLURM_BIN` wrong, Slurm down, **or** an old `<job>.kernels` folder ([known issue](../../known-issues/jupyter.md#the-page-can-break-on-a-leftover-kernels-folder)) | fix `SLURM_BIN`; or delete old `~/.jupyter-sessions/*.kernels` folders |
| Page: `unknown choice '<x>'` | an old page after a GPU change | reload |
