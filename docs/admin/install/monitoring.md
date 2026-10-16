# 9 · Monitoring

> One Prometheus on master collects every node's hardware, GPU, Slurm and service state, evaluates alerts, and
> feeds the read-only, admin-only **Cluster monitor** page in the portal.

**Folder:** [`OOD-Install/9-monitoring`](https://github.com/Ankush610/OOD-Install/tree/main/9-monitoring) ·
**Explained in:** [Monitoring](../../how-it-works/12-monitoring.md) ·
**Metrics and alerts:** [Monitoring reference](../../reference/monitoring.md) ·
**Using the page:** [Cluster monitor](../cluster-monitor.md) ·
**Known issues:** [Monitoring](../../known-issues/monitoring.md)

## What it sets up

- **Prometheus** (3.5 LTS) on master at `PROM_URL`, under systemd (not Kubernetes), with 30 days of history.
- **node_exporter** on every node, watching hardware and the core services in `MONITOR_UNITS`.
- **Two collectors:** GPUs from `nvidia-smi` every 15 s (GPU nodes), Slurm from `scontrol`/`squeue --json` every
  30 s (master). No GPU operator, no `slurm.conf` change.
- **blackbox_exporter** on master, which tries six services the way a user would: portal, LDAP, Keycloak, MLflow,
  registry, gateway.
- **Named queries** (recording rules) for apps, and **34 alert rules**. No email yet.
- **The Cluster monitor** portal app, readable only by `OOD_ADMIN`.

## Before you start

| Script | Runs on | Root |
|---|---|---|
| `1-install.sh` | master (sets up every node over root SSH) | yes |
| `2-install-ood-app.sh` | login node | yes |

Checked, never installed or changed:

- Slurm answers on master (`sinfo`, `scontrol show config`).
- Root SSH from master to every node. Unreachable nodes are skipped, not fatal.
- `/home` mounted on every node (nodes install from `MONITOR_ROOT`).
- The NVIDIA driver on nodes where Slurm lists GPUs (otherwise a warning and `gpu_collector_ok 0`).
- `PROM_PORT` (9090) free on master.
- firewalld: rules are added only if it runs.

Internet on master **once**, for three tarballs from GitHub (Prometheus, node_exporter, blackbox_exporter). Each is
checked against the sha256 in `site.conf`. **Offline:** copy all three tarballs into `MONITOR_ROOT` and rerun.

For the page: [Portal](portal.md), [LDAP](ldap.md), and the `5-model-hub` folder in the checkout (for the look).
The six probes expect registry, LDAP, portal, Keycloak, MLflow and the gateway; any that isn't installed shows as
down.

## Settings used

| Variable | Example | Meaning |
|---|---|---|
| `PROMETHEUS_VERSION` / `_SHA256` | `3.5.5` / hash | change both together (from the release's `sha256sums.txt`) |
| `NODE_EXPORTER_VERSION` / `_SHA256` | `1.12.1` / hash | also the node-mixin version the node alerts come from |
| `BLACKBOX_VERSION` / `_SHA256` | `0.28.0` / hash | |
| `MONITOR_ROOT` | `/home/apps/monitoring` | releases and collectors on the shared `/home` |
| `PROM_DATA` | `/var/lib/prometheus` | the database, on master's **local** disk |
| `PROM_PORT`, `PROM_URL` | `9090`, `http://$MASTER_IP:9090` | firewalld lets in only `LOGIN_IP` |
| `PROM_RETENTION`, `PROM_RETENTION_SIZE` | `30d`, `20GB` | whichever comes first |
| `NODE_EXPORTER_PORT`, `BLACKBOX_PORT` | `9100`, `9115` | node_exporter from `MASTER_IP` only; blackbox on 127.0.0.1 |
| `MONITOR_UNITS` | `slurmctld\|slurmd\|…\|nfs-server` | services watched by `ServiceDown` (each node watches the ones it has **enabled**) |
| `RUNBOOK_URL` | the 9-monitoring README | where "How to fix" links go (`#<alertname in lower case>`) |

## Install

### 1. Prometheus, exporters, collectors (master)

```bash
sudo bash 9-monitoring/1-install.sh
```

| Step | What it does |
|---|---|
| `== 1. Checks` | installs `curl`, `python3`, `tar` if missing; stops if Slurm doesn't answer or 9090 is taken; finds GPU nodes from `sinfo`; warns about GPU nodes missing from `site.conf` |
| `== 2. Releases + collectors` | downloads (or uses hand-copied) tarballs, checks sha256, unpacks into `MONITOR_ROOT`, copies the collectors |
| `== 3. Every node` | runs `node-setup.sh` on each node: node_exporter + service watch, the GPU collector on GPU nodes, the Slurm collector on master, the firewall rule |
| `== 4. Prometheus` | creates the `prometheus` user and `PROM_DATA`; sets up blackbox; **tests the rules first** (a failure changes nothing); writes config and rules; restarts or reloads only if needed; adds the firewall rule; waits for `/-/ready` |
| `== 5. Check` | after 20 s prints each node's exporter, CPU, RAM, Slurm state and GPUs, then the collector status |

**Expect:** every node `up` and `collectors: all ok`, then the API, web UI and service-check URLs (plus
`SKIPPED (unreachable): …` if a node was down). CPU reads too high for the first 5 minutes (it is a 5-minute rate).

```bash
# check from master or the login node
curl -s "$PROM_URL/-/ready"
curl -s "$PROM_URL/api/v1/query" --data-urlencode 'query=slurm:node_gpus_free'
```

### 2. The Cluster monitor (login node)

```bash
sudo bash 9-monitoring/2-install-ood-app.sh
```

It checks Prometheus answers **from the login node** (so the firewall is right), copies the app with Model Hub's
look, writes `site.json`, restricts the folder to `OOD_ADMIN`'s group (750/640), then checks an LDAP user can't read
it and prints the node and GPU counts as `OOD_ADMIN` sees them.

Log in as `OOD_ADMIN`, **Restart Web Server**, open **Interactive Apps → Cluster monitor** (or the card in the home
page's Admin section).

## Everyday tasks

| Task | How (master, root unless noted) |
|---|---|
| Open Prometheus' own web UI | browse `PROM_URL` from the login node, or `ssh -L 9090:<master-ip>:9090 …` |
| See firing alerts | the Cluster monitor's Problems box, `$PROM_URL/alerts`, or `curl -s $PROM_URL/api/v1/alerts` |
| Add a node | add it to `COMPUTE_NODES`, rerun `1-install.sh` (a GPU node also gets the GPU collector) |
| A node was skipped | rerun `1-install.sh` when it is back |
| Watch a newly enabled service | enable the unit, make sure it matches `MONITOR_UNITS`, rerun `1-install.sh` |
| Upgrade a component | change `*_VERSION` **and** `*_SHA256` together, rerun |
| Change an alert or rule | edit `9-monitoring/rules.yml` (add a test to `rules_test.yml`), rerun. **Never** edit `/etc/prometheus/*`: the next run overwrites it |
| Add a node-mixin alert | edit `node-alerts.jsonnet`, re-render with go-jsonnet (see its header), commit `node-alerts.yml` |
| Add a runbook row | every alert needs an `<a id="<alertname lowercase>"></a>` row in the README's "Alerts: what to do", or `1-install.sh` warns |
| Run a collector by hand | `/usr/local/libexec/aistack-monitoring/gpu-metrics` or `…/slurm-metrics` (prints to the screen) |
| Test locally | `python3 9-monitoring/slurm-metrics.py --test`; `promtool check rules …`; `promtool test rules 9-monitoring/rules_test.yml` |
| Update the Cluster monitor | rerun `2-install-ood-app.sh`, then Restart Web Server |
| Planned maintenance | drain the node in Slurm. There is no alert silencing until email alerts exist |
| Stop or start Prometheus | `systemctl stop\|start prometheus` (the data stays) |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `1-install.sh`: Slurm doesn't answer | `slurmctld` or munge down, or `SLURM_BIN` wrong | `systemctl status slurmctld munge`; `which sinfo` |
| `Port 9090 is taken by another program` | something else on `PROM_PORT` | stop it, or change `PROM_PORT` |
| `No internet here. Download <url> …` | master can't reach GitHub | download elsewhere, copy to `MONITOR_ROOT/<release>.tar.gz`, rerun |
| Checksum doesn't match | `*_VERSION` changed without `*_SHA256` | copy the hash from the release's `sha256sums.txt` |
| `WARNING: Slurm has GPUs on <h>, which isn't in site.conf` | GPU node missing from `COMPUTE_NODES` | add it, rerun |
| `SKIPPED <node>: no …/node_exporter-… here` | `/home` not mounted there | mount it, rerun |
| `SKIPPED <node>: can't install curl/python3` | the node's package repos are unreachable | fix them or install by hand, rerun |
| `SKIPPED <node>: node_exporter doesn't answer on :9100` | the exporter failed to start | `journalctl -u node_exporter` on that node |
| `WARNING: Slurm says GPUs, but no nvidia-smi` | no NVIDIA driver | install it |
| `WARNING: firewalld is off …: port <p> is open to the whole network` | firewalld inactive | start it if your policy requires, rerun |
| `Rules rejected (promtool test rules …): nothing changed` | a rule is broken or a test fails | `promtool test rules 9-monitoring/rules_test.yml`; the old rules keep running |
| `WARNING: no row in README.md "Alerts: what to do" for: X` | new alert without a runbook row | add the anchor row |
| `blackbox_exporter rejects …/blackbox.yml` / `promtool rejects …` | generated config invalid | check versions and odd `site.conf` values |
| `Prometheus not ready after 1 min` / won't start | bad config or rules | `promtool check config /etc/prometheus/prometheus.yml`; `journalctl -u prometheus -n 50` |
| `node` target down: `no route to host` | node off or unreachable | bring it back, rerun |
| `node` target down: `connection refused` | node_exporter not running | `systemctl status node_exporter` there; rerun |
| `node` target down: `context deadline exceeded`, node fine | the node's firewall blocks `MASTER_IP` | rerun (it adds the rule) |
| `gpu_collector_ok 0` / `CollectorFailing` | `nvidia-smi` missing or failing | `ssh <node> nvidia-smi`; `journalctl -u gpu-metrics -n 20` |
| `slurm_collector_ok 0` | `scontrol`/`squeue` fail as `node_exporter` | `journalctl -u slurm-metrics -n 20`; `sudo -u node_exporter squeue` |
| `CollectorStale` | timer stopped, or textfile folder not writable | `systemctl list-timers gpu-metrics.timer slurm-metrics.timer` |
| GPU count lower than Slurm's | a GPU node is down, or the card name doesn't contain Slurm's type (`gpu:a30` → "NVIDIA A30") | match Slurm's gres type to the card name |
| `2-install-ood-app.sh`: `Prometheus doesn't answer at PROM_URL from LOGIN_HOST` | step 1 not run, or the firewall blocks `LOGIN_IP` | run step 1; check the rich rule |
| `No …/5-model-hub/…/style.css` | incomplete checkout | use the whole OOD-Install folder |
| `WRONG: <user> can read …/cluster_status` | `OOD_ADMIN`'s primary group includes other users | give `OOD_ADMIN` a private group, rerun |
| Page: "monitoring doesn't answer" | Prometheus stopped, or firewall | `systemctl status prometheus`; `curl $PROM_URL/-/ready` from the login node |
| Page: "monitoring has no Slurm data" | the Slurm collector isn't running on master | `systemctl list-timers slurm-metrics.timer`; `journalctl -u slurm-metrics -n 20` |
| Page: wrong GPU shown as reserved | `gres.conf` lists GPUs in a different order than the PCI bus | list them in bus order (or `AutoDetect=nvml`) |
| Page not in Interactive Apps | the app list is cached | Restart Web Server |
| No Cluster monitor card on the home page | the card list is fixed in the portal's widget | rerun `2-ood/setup-ood.sh`, Restart Web Server |
| Services tab: "Service checks aren't running" | blackbox_exporter down | `systemctl status blackbox_exporter` |
| Services tab: "No node reports its services yet" | step 1 not run, or no watched unit enabled | run `1-install.sh` |
| A component you didn't install shows Down, `ServiceUnreachable` fires | the six probes are fixed | install it, or remove the target from `1-install.sh` ([known issue](../../known-issues/monitoring.md#probes-are-fixed-for-all-six-services)) |
