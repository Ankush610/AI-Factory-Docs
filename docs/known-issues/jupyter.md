# Jupyter

Issues in [`7-jupyter`](https://github.com/Ankush610/OOD-Install/tree/main/7-jupyter) and in the page it shares with
VS Code. How to read this page: [Known issues](index.md#how-to-read-these-pages).

| Issue | Type | Confidence |
|---|---|---|
| [The page can break on a leftover kernels folder](#the-page-can-break-on-a-leftover-kernels-folder) | Bug | Inferred |
| [Session token travels in the URL](#session-token-travels-in-the-url) | Security | Inferred |
| [Sessions are protected only by their token](#sessions-are-protected-only-by-their-token) | Security | Inferred |
| [Kernel files are written without escaping](#kernel-files-are-written-without-escaping) | Bug | Confirmed |
| [Idle stop takes longer than the setting](#idle-stop-takes-longer-than-the-setting) | Docs | Inferred |
| [Apptainer and the training images come from outside the repo](#apptainer-and-the-training-images-come-from-outside-the-repo) | Gap | Confirmed |
| [Smaller items](#smaller-items) | Mixed | — |

## The page can break on a leftover kernels folder

- **Type:** Bug · **Confidence:** Inferred (not tested) · **Where:** `4-vscode/ood-app/vscode/passenger_wsgi.py:94-99,177-181`,
  `7-jupyter/ood-app/jupyter/job.sh:20,29`
- **What happens:** each Jupyter session creates a **folder** `~/.jupyter-sessions/<job>.kernels/`, removed only by
  the job's exit trap. If the trap doesn't run (killed after Slurm's grace time, node crash), the folder stays. After
  7 days the page's cleanup calls `os.remove()` on it, which fails on a folder. The page then shows "Could not ask
  Slurm for your sessions" with an empty list on **every** load. VS Code isn't affected.
- **Workaround:** `rm -rf ~/.jupyter-sessions/*.kernels` for that user (only when they have no running session).
- **Fix:** use `shutil.rmtree` for folders in `prune()`.

## Session token travels in the URL

- **Type:** Security · **Confidence:** Inferred · **Where:** `4-vscode/ood-app/vscode/passenger_wsgi.py:37,144-147`
- **What happens:** **Connect** opens `/node/<host>/<port>/lab?token=<token>`. The token can land in Apache's access
  log on the login node and in browser history, and stays valid for the whole session (up to 12 hours). This design
  keeps Jupyter's XSRF protection on (a posted password would need `_xsrf`).

## Sessions are protected only by their token

- **Type:** Security · **Confidence:** Inferred · **Where:** `7-jupyter/ood-app/jupyter/job.sh:36`, `2-ood/setup-ood.sh:127-129`
- **What happens:** Jupyter listens on `0.0.0.0` with no firewall rule, and the portal's `/node` proxy has no
  `host_regex`. Anyone who reaches the port, or any portal user through `/node`, reaches Jupyter; the token is the
  only barrier. The hop from Apache to the node is plain HTTP.

## Kernel files are written without escaping

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `7-jupyter/ood-app/jupyter/job.sh:25-26`
- **What happens:** `kernel.json` is built with `printf`. An image path containing `"` or `\` produces broken JSON
  and that kernel won't load.
- **Workaround:** keep image names to letters, digits, `.`, `-` and `_`.

## Idle stop takes longer than the setting

- **Type:** Docs · **Confidence:** Inferred · **Where:** `7-jupyter/ood-app/jupyter/job.sh:42-44`, `site.conf:158`
- **What happens:** the same value is used for culling idle kernels (checked every 300 s) and for the server's
  no-activity shutdown. The real time to stop can be up to idle + 300 s for the kernels, then the idle time again for
  the server. Confirm the worst case before telling users "1 hour".

## Apptainer and the training images come from outside the repo

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `site.conf:108-110`, `7-jupyter/README.md:33`
- **What happens:** `APPTAINER`, `CONTAINERS_ROOT`, `/home/apps/containers/README.md` and `apptainer.sh` are
  referenced but not in OOD-Install. The rule "every training image has `ipykernel`" has no check here; see
  [OOD-Container](https://github.com/Ankush610/OOD-Container).

## Smaller items

- `1-install-jupyter.sh` checks root but not the host. (Docs, Confirmed)
- The page's docstring says Connect "posts that password"; for Jupyter it is a GET with a token. (Docs, Confirmed)
- Session ports fall in 20000–52767, not up to 59999 (`$RANDOM` stops at 32767). (Bug, Confirmed)
- The upgrade steps give no command for the scratch-venv freeze, and nothing shows when an old install folder is
  unused. (Docs, Confirmed)
- The top-level `README.md` folder table and install block stop at `4-vscode`. (Docs, Confirmed)
