# 4 · VS Code in the browser

> Every portal user gets their own VS Code (code-server) session, started as a Slurm job that runs as them: on a
> shared `viewer` partition on the login node, or on a GPU node.

**Folder:** [`OOD-Install/4-vscode`](https://github.com/Ankush610/OOD-Install/tree/main/4-vscode) ·
**Explained in:** [VS Code and Jupyter](../../how-it-works/07-vscode-and-jupyter.md) ·
**Known issues:** [VS Code](../../known-issues/vscode.md)

## What it sets up

- **code-server, installed once** on the shared `/home` (`CODE_SERVER_ROOT/<version>` and a `current` link), so
  every node runs the same copy.
- **The `viewer` partition** on the login node: Slurm node `viewer01` (`NodeHostname=<login node>`), up to
  `VIEWER_OVERSUBSCRIBE` sessions per core, with a fixed share of cores and memory kept out of Slurm for SSH users
  and services.
- **The portal page** **Interactive Apps → VS Code**: a custom page (not an Open OnDemand form) that runs as the
  user and calls `sbatch`, `squeue` and `scancel` itself. Launch, status, Connect and Stop are on one page.
- **GPU choices** read from Slurm at install time: "GPU node: n × <type>", with n GPUs and n shares of the node's
  CPUs and memory.

The same page also serves [Jupyter](jupyter.md).

## Before you start

| Script | Runs on | Root |
|---|---|---|
| `1-install-code-server.sh` | master (any node that writes the shared `/home` works) | yes |
| `2-slurm-viewer.sh` | master (where `slurmctld` runs) | yes |
| `3-install-ood-app.sh` | login node | yes |

- **Done first:** [LDAP](ldap.md), [Portal](portal.md).
- The login node has **munge and `slurmd`** installed (it runs the sessions). The script checks both.
- Root SSH from master to the login node and every compute node.
- `python3` and `envsubst` (gettext) on the login node.
- Internet on master: the code-server release from GitHub (about 220 MB). Session nodes need internet to install
  extensions from Open VSX; without it users install `.vsix` files.
- **For the page's look:** the [Model Hub](model-hub.md) portal app. The page borrows its styles, and stays
  unstyled until Model Hub is installed. The installer only warns.

## Settings used

| Variable | Example | Meaning |
|---|---|---|
| `CODE_SERVER_VERSION`, `CODE_SERVER_ROOT` | `4.139.1`, `/home/apps/code-server` | release and shared install folder |
| `VIEWER_PARTITION`, `VIEWER_NODE` | `viewer`, `viewer01` | the partition and the login node's **Slurm** name (never its hostname) |
| `VIEWER_RESERVED_CORES`, `VIEWER_RESERVED_MEM_MB` | `8`, `49152` | kept out of Slurm (`CoreSpecCount`, `MemSpecLimit`) |
| `VIEWER_OVERSUBSCRIBE` | `4` | sessions per core |
| `VSCODE_MEM` | `2G` | memory per "Editor only" session |
| `VSCODE_IDLE_SECONDS` | `3600` | an editor with no browser attached stops after this |
| `GPU_PARTITION` | `auto` | which partition offers GPU sessions; `""` = none |
| `SLURM_BIN`, `SLURM_CONF` | `/usr/local/bin`, `/etc/slurm/slurm.conf` | where Slurm is |

## Install

### 1. code-server (master)

```bash
sudo bash 4-vscode/1-install-code-server.sh
```

Downloads the release (resuming a dropped download, up to 5 tries, until `gzip -t` passes), unpacks it into
`CODE_SERVER_ROOT/<version>`, points `current` at it, then prints the version locally and on each compute node
through `srun`.

**Expect:** a version line such as `4.139.1` for master and each compute node. `(srun failed, check by hand)` means
Slurm couldn't run the check on that node.

### 2. The viewer partition (master)

```bash
sudo bash 4-vscode/2-slurm-viewer.sh
```

| Step | What it does |
|---|---|
| `== 1.` | checks munge and `slurmd` on the login node; runs `slurmd -C` there; takes 98% of its memory; builds the node line (`NodeName=viewer01 NodeHostname=<login> … CoreSpecCount MemSpecLimit`) and the partition line (`MaxTime=12:00:00 OverSubscribe=FORCE:4`) |
| `== 2.` | backs up `slurm.conf`, replaces or adds both lines (an older `NodeName=<master>` line is replaced) |
| `== 3.` | copies `slurm.conf` to the login node and every compute node (unreachable ones are `SKIPPED`) |
| `== 4.` | restarts `slurmctld` and the login node's `slurmd` only if the node's resources changed; then `scontrol reconfigure` |
| `== 5.` | shows the node and `sinfo -p viewer`, runs a test job, removes slurm-bridge's taint from master if present |

**Expect:** the test `srun` prints the login node's hostname, and `sinfo` shows `FORCE:4`.

### 3. The portal page (login node)

```bash
sudo bash 4-vscode/3-install-ood-app.sh
```

It checks the portal, code-server and the `viewer` partition exist; detects the GPU partition and sizes; replaces
`/var/www/ood/apps/sys/vscode` with a fresh copy, fills `job.sh`, writes `site.json`; then syntax-checks the result.

**Expect:** `job.sh: OK`, `passenger_wsgi.py: OK`, and a `site.json` with `"app": "vscode"`. Click
**Restart Web Server**; **Interactive Apps → VS Code** opens the page.

!!! note "Never edit the installed copy"
    Edit `4-vscode/ood-app/` in the repo and rerun the installer. Each run deletes and recopies the app folder.

## Everyday tasks

| Task | How |
|---|---|
| Upgrade code-server | set `CODE_SERVER_VERSION`, rerun `1-install-code-server.sh`. Running sessions keep the old version until they end |
| Remove old versions | delete `CODE_SERVER_ROOT/<old version>` once no session uses it |
| Change session memory or idle time | edit `VSCODE_MEM` / `VSCODE_IDLE_SECONDS`, rerun `3-install-ood-app.sh`, Restart Web Server |
| Change the login node split | edit `VIEWER_RESERVED_*`, `VIEWER_OVERSUBSCRIBE`, rerun `2-slurm-viewer.sh` (running jobs survive) |
| GPU nodes added or changed | rerun `3-install-ood-app.sh` **and** `7-jupyter/2-install-ood-app.sh`, Restart Web Server |
| Changed the page code | rerun both installers (VS Code and Jupyter each install their own copy) |
| A node missed the `slurm.conf` copy | rerun `2-slurm-viewer.sh`, or `scp $SLURM_CONF root@<node>:$SLURM_CONF` |
| See every VS Code session | `squeue -n vscode` |
| See viewer load | `squeue -p viewer`; `sinfo -p viewer -o "%P %N %c %m %h %l"` |
| Stop someone's session | Admin page → Running → Stop (the user sees a notice), or `scancel <job>` |
| Resume a drained viewer node | rerun `2-slurm-viewer.sh`, then `scontrol update nodename=viewer01 state=resume` |
| Read a session's log | `~<user>/.vscode-sessions/<job>.log` |

**Capacity (test cluster):** the login node has 48 cores and 192 GB. Slurm gets 40 cores and about 135 GB, which
fits 160 sessions by CPU but about **67 by memory** (2 GB each). Memory is the real limit.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `unexpected end of file` / `Download failed` | the download dropped | rerun (it resumes) |
| Session stays **Waiting** | `viewer` is out of memory, or no GPU is free | `squeue -p viewer`; lower `VSCODE_MEM` or the reservation, or wait |
| `slurmd -C on <login> says: …` / `systemctl cat slurmd` fails | no root SSH to the login node, or no `slurmd` there | install Slurm + munge on the login node; test `ssh root@<login> slurmd -C` |
| `viewer01` drained, `Low RealMemory` | `RealMemory` above what the kernel reports | rerun `2-slurm-viewer.sh`, then `scontrol update nodename=viewer01 state=resume` |
| `Reservation (…) leaves nothing of …` | the reservation is larger than the node | lower `VIEWER_RESERVED_*` |
| GPU choices wrong after GPU nodes changed | choices are fixed at install | rerun both app installers, Restart Web Server |
| `partition '<x>' has no GPUs` | `GPU_PARTITION` names a partition without GPUs | set `auto`, a listed partition, or `""` |
| `sbatch` from VS Code's terminal fails, works from SSH | the session's own Slurm variables leaked into the new job | `job.sh` clears them now; rerun `3-install-ood-app.sh`, start a new session |
| Cluster DNS down; coredns, KEDA… **Pending** (`untolerated taint`) | master's Slurm node is named like its Kubernetes node, so slurm-bridge taints it | set `VIEWER_NODE` (not the hostname), rerun `2-slurm-viewer.sh` |
| code-server exits at once, printing nothing | `VSCODE_IPC_HOOK_CLI` is set | `job.sh` already unsets it |
| Session starts, then disappears; log says `code-server did not start` | wrong `CODE_SERVER_ROOT`, a crash, or no answer within 120 s | `<CODE_SERVER_ROOT>/current/bin/code-server --version` on that node |
| Page: `Could not ask Slurm for your sessions: …` | `SLURM_BIN` isn't the folder of `squeue` | fix it, rerun `3-install-ood-app.sh` |
| **Connect** shows code-server's login page | the session password didn't reach code-server | relaunch |
| `slurmd: command not found` under `sudo` | `sudo` drops `/usr/local/bin` | set `SLURM_BIN` to the folder of `sbatch` |
| `SKIPPED <node> (unreachable)` | no root SSH to that node | copy `SLURM_CONF` there by hand (it must match everywhere), rerun |
| `No /var/www/ood/apps/sys` / `No code-server` / `No partition viewer` | an earlier step is missing (or `/home` isn't mounted) | run the missing step |
| `unfilled values above` | a new placeholder in `job.sh` isn't in the `envsubst` list | add it to the list and the `export` |
| `WARNING: no Model Hub app` | Model Hub's portal app isn't installed yet | install it; the page works but is unstyled until then |
| Page: `unknown choice 'gpuN'` / `no VS Code session <id>` | the page is older than `site.json`, or the session already ended | reload the page |
| Extension missing | not on Open VSX | install a `.vsix` (Extensions → … → Install from VSIX) |
| Old sessions in **My Interactive Sessions** show errors | started by the old form, which is gone | they run until their hours end; `scancel` ends them now |
