# Monitoring

Issues in [`9-monitoring`](https://github.com/Ankush610/OOD-Install/tree/main/9-monitoring). How to read this page:
[Known issues](index.md#how-to-read-these-pages).

| Issue | Type | Confidence |
|---|---|---|
| [No notifications, and no watch on the watcher](#no-notifications-and-no-watch-on-the-watcher) | Gap | Confirmed |
| [Probes are fixed for all six services](#probes-are-fixed-for-all-six-services) | Bug | Confirmed |
| [GPU health can vanish silently on some drivers](#gpu-health-can-vanish-silently-on-some-drivers) | Bug | Inferred |
| [Runbook links need internet access](#runbook-links-need-internet-access) | Gap | Confirmed |
| [README leaves out blackbox_exporter](#readme-leaves-out-blackbox_exporter) | Docs | Confirmed |
| ["Model Hub reads Prometheus" isn't true yet](#model-hub-reads-prometheus-isnt-true-yet) | Docs | Confirmed |
| [Smaller items](#smaller-items) | Mixed | — |

## No notifications, and no watch on the watcher

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `9-monitoring/README.md:176`, `cluster-status-improvement.md:65-77`
- **What happens:** there is no Alertmanager, so nothing emails or pages anyone; alerts show only in the Cluster
  monitor and at `$PROM_URL/alerts`. Prometheus on master is a single point: if master dies, monitoring dies
  silently. Email, silences and an outside heartbeat are "Phase 2", waiting on SMTP details.

## Probes are fixed for all six services

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `9-monitoring/1-install.sh:174-179`
- **What happens:** the portal, LDAP, Keycloak, MLflow, registry and gateway are always probed. A site without one of
  them gets a permanent critical `ServiceUnreachable`. There is no skip list.
- **Workaround:** remove the target from `1-install.sh` (and expect it back on the next pull).
- **Related (inferred):** the `TargetDown` summary uses `$labels.node`, which blackbox targets don't have, so its text
  shows an empty node name.

## GPU health can vanish silently on some drivers

- **Type:** Bug · **Confidence:** Inferred · **Where:** `9-monitoring/gpu-metrics.sh:13`
- **What happens:** the code asks `nvidia-smi` for `clocks_event_reasons.hw_slowdown`; the plan says
  `clocks_throttle_reasons.hw_slowdown` was the verified name. On a driver that doesn't know the field, the **whole**
  health line (ECC, remap, slowdown) is lost, and errors go to `/dev/null`.
- **Check:** `nvidia-smi --query-gpu=clocks_event_reasons.hw_slowdown --format=csv` on each GPU node.

## Runbook links need internet access

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `site.conf:144`
- **What happens:** `RUNBOOK_URL` points to this site's Monitoring reference on GitHub Pages. On an air-gapped
  site "How to fix" links are dead (host a copy of the site locally and point `RUNBOOK_URL` at it). The anchors must stay `#<alertname in lower case>`, and
  `#alerts-what-to-do` is also linked.

## README leaves out blackbox_exporter

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `9-monitoring/README.md:7-33,50`
- **What happens:** the file table, diagram and "Installs" row don't mention it, and the "offline site" step lists
  only two tarballs. The install also needs `blackbox_exporter-<version>.linux-amd64.tar.gz`.

## "Model Hub reads Prometheus" isn't true yet

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `9-monitoring/README.md:3`, `site.conf:131`
- **What happens:** both say Model Hub reads `PROM_URL`. Nothing outside `9-monitoring` uses it; "GPUs free now" in
  Model Hub is still planned (Phase 4).

## Smaller items

- **`/home` at 90 %** (Gap, Confirmed): the checklist item is still open. Node alerts fire only below 5 % free, or
  when filling is predicted, and only for `ext4|xfs|btrfs`.
- **Hidden Slurm jobs** (Bug, Inferred): the Slurm collector runs as `node_exporter`. With `PrivateData=jobs` in
  `slurm.conf`, job and user counts would be empty.
- **Admin-only depends on the primary group** (Security, Inferred): same as the [Admin page](admin-page.md#visibility-depends-on-the-admins-primary-group).
- **Removing a node** (Gap, Confirmed): taking it out of `COMPUTE_NODES` stops scraping, but its exporter and
  collectors stay installed. No procedure.
- **No backup for `PROM_DATA`** (Gap, Confirmed): is 30 days of history disposable?
- **Service UIDs not pinned** (Gap, Confirmed): `prometheus` and `node_exporter` get automatic system IDs, unlike the
  repo's fixed service IDs.
- **Plan document is partly stale** (Docs, Confirmed): `cluster-status-improvement.md`'s "Gaps" table describes the
  state before Phase 1.
- **Restart wording** (Docs, Confirmed): "Help → Restart Web Server" here, "top-right menu" in the top-level README.
- **Partial rule tests** (Gap, Confirmed): `TargetDown`, `Collector*`, `GpuThrottled`, `GpuHot` and `Prometheus*`
  have no tests.
