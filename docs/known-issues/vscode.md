# VS Code

Issues in [`4-vscode`](https://github.com/Ankush610/OOD-Install/tree/main/4-vscode). How to read this page:
[Known issues](index.md#how-to-read-these-pages).

| Issue | Type | Confidence |
|---|---|---|
| [`CLAUDE.md` describes an app that no longer exists](#claudemd-describes-an-app-that-no-longer-exists) | Docs | Confirmed |
| [Sessions are protected only by their password](#sessions-are-protected-only-by-their-password) | Security | Inferred |
| [No checksum for the code-server download](#no-checksum-for-the-code-server-download) | Security | Confirmed |
| [Page is unstyled until Model Hub is installed](#page-is-unstyled-until-model-hub-is-installed) | Gap | Inferred |
| [cgroup limits are assumed, not set](#cgroup-limits-are-assumed-not-set) | Gap | Confirmed |
| [GPU partition time limit isn't checked](#gpu-partition-time-limit-isnt-checked) | Gap | Confirmed |
| [Session ports never reach the top of the range](#session-ports-never-reach-the-top-of-the-range) | Bug | Confirmed |
| [README and doc mismatches](#readme-and-doc-mismatches) | Docs | Confirmed |
| [Open questions](#open-questions) | Question | — |

## `CLAUDE.md` describes an app that no longer exists

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `CLAUDE.md:32,47,63`
- **What happens:** it calls VS Code a "Batch Connect" app and describes `form.yml`, `submit.yml.erb`,
  `view.html.erb` and `bc_dynamic_js`. None of these files exist; the app is a Passenger page
  (`passenger_wsgi.py`). Its local sanity check loads the missing `form.yml`, so it fails.

## Sessions are protected only by their password

- **Type:** Security · **Confidence:** Inferred · **Where:** `4-vscode/ood-app/vscode/job.sh:23`, `2-ood/setup-ood.sh:120-129`
- **What happens:** code-server listens on `0.0.0.0:<random port>` with no firewall rule, and the portal's proxy has
  no `host_regex` set. Anyone who can reach the port (or any portal user through `/rnode`) reaches the code-server
  login page. The random per-session password is the only barrier.
- **Workaround:** limit ports 20000–59999 on the session nodes to `LOGIN_IP` with a firewall rule.

## No checksum for the code-server download

- **Type:** Security · **Confidence:** Confirmed · **Where:** `4-vscode/1-install-code-server.sh:26-29`
- **What happens:** the tarball is checked only with `gzip -t`. Monitoring pins SHA-256 values for its downloads;
  there is no `CODE_SERVER_SHA256`.

## Page is unstyled until Model Hub is installed

- **Type:** Gap · **Confidence:** Inferred · **Where:** `4-vscode/ood-app/vscode/passenger_wsgi.py:48,189-191`
- **What happens:** the page loads Bootstrap and styles from `/pun/sys/model_hub/`, but the install order puts
  VS Code before Model Hub. The installer only warns.

## cgroup limits are assumed, not set

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `4-vscode/README.md:51`
- **What happens:** the README says `ConstrainCores` and `ConstrainRAMSpace` in `cgroup.conf` hold each session to
  what it asked for. No script sets or checks them. Without them, one session can use more than its share.
- **Workaround:** check `/etc/slurm/cgroup.conf` on the login and GPU nodes.

## GPU partition time limit isn't checked

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `4-vscode/ood-app/vscode/passenger_wsgi.py:103`
- **What happens:** the page allows up to 12 hours for GPU sessions, but only the viewer partition's `MaxTime` is
  set. If the GPU partition's limit is shorter, the job is rejected or waits forever.

## Session ports never reach the top of the range

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `4-vscode/ood-app/vscode/job.sh:15-17` (same in Jupyter)
- **What happens:** `$RANDOM` goes up to 32767, so ports fall in 20000–52767, not up to 59999. Harmless, but the
  docs and firewall rules should use the real range.

## README and doc mismatches

- **Type:** Docs · **Confidence:** Confirmed
- `4-vscode/README.md:13` refers to "the Slurm note above", which isn't there; lines 17–18 are an empty code block.
- The top-level `README.md:29` says the viewer partition is "on master"; the code puts it on the login node.
- `1-install-code-server.sh` has no host check, and its `srun -w <node>` uses the default partition, which fails if
  the compute nodes aren't in it.
- These docs used to say a ready session shows "Running"; the page shows **Ready** (and **Starting** while
  code-server comes up). Fixed in the [How it works](../how-it-works/07-vscode-and-jupyter.md) page.

## Open questions

- `job.sh` relies on Slurm setting `$HOME` and `$SLURM_JOB_ID` under `sbatch --export=NONE`; the repo doesn't say so.
- The idle stop "with no browser attached" depends on code-server 4.139.1's `--idle-timeout-seconds` behaviour,
  which the repo doesn't confirm.
- There is no way to tell which `CODE_SERVER_ROOT/<version>` running sessions still use.
