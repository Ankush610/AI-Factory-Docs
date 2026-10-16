# Monitoring reference

Metrics, named queries and alerts provided by [9-monitoring](../admin/install/monitoring.md). For what to do when
an alert fires, the Cluster monitor's **How to fix** links go to the runbook:
[Alerts: what to do](https://github.com/Ankush610/OOD-Install/blob/main/9-monitoring/README.md#alerts-what-to-do).

## Scrape jobs

Every 15 s, external label `cluster: <CLUSTER_ID>`.

| Job | Targets | Notes |
|---|---|---|
| `prometheus` | Prometheus itself | for the `Prometheus*` alerts |
| `node` | `<host>:9100` on every node | `node` label = hostname; `honor_labels: true` keeps the Slurm collector's own `node` label |
| `blackbox` | 6 service URLs (below) | each labelled `service` and `module` |

## Service probes

| `service` | Target | Module |
|---|---|---|
| `ood` | `https://<LOGIN_HOST>/` | `http_up`: 5 s timeout; 200 or a redirect counts as up |
| `ldap` | `<master>:636` | `tls_up`: TCP + TLS handshake |
| `keycloak` | `$KEYCLOAK_URL/realms/$KEYCLOAK_REALM` | `http_up` |
| `mlflow` | `$MLFLOW_URI/health` | `http_up` |
| `registry` | `http://$REGISTRY/v2/` | `http_up` |
| `gateway` | `$GATEWAY_URL/healthz` | `http_up` |

Probes skip certificate verification (the certificates are self-signed) but still measure certificate expiry.

## Raw metrics from the collectors

All gauges. GPU series carry `node, gpu, name, uuid`; Slurm node series carry `node` (the hostname) and
`slurm_node` (Slurm's own name, e.g. `viewer01`).

| Metric | Meaning |
|---|---|
| `gpu_utilization_ratio` | GPU busy, 0–1 |
| `gpu_memory_used_bytes`, `gpu_memory_total_bytes` | GPU memory |
| `gpu_temperature_celsius` | temperature |
| `gpu_power_watts` | power draw (absent on cards that report `[N/A]`) |
| `gpu_ecc_errors_uncorrected` | uncorrectable memory errors since the driver loaded |
| `gpu_remapped_rows_pending` | 1 = a memory row waits for a GPU reset |
| `gpu_hw_slowdown` | 1 = the hardware cut clocks (power brake or heat) |
| `gpu_collector_ok` | 1 = `nvidia-smi` worked |
| `slurm_node_info` | always 1; labels `state, reason, partitions, gpu_type` |
| `slurm_node_available` | 1 = takes new jobs |
| `slurm_node_cpus`, `slurm_node_cpus_alloc` | CPUs jobs can get / reserved now |
| `slurm_node_memory_bytes`, `slurm_node_memory_alloc_bytes` | memory jobs can get / reserved now |
| `slurm_node_gpus`, `slurm_node_gpus_alloc` | GPUs / in use |
| `slurm_node_gpu_reserved` | 1 per reserved GPU (label `gpu` = Slurm index) |
| `slurm_jobs` | running/pending job counts by `state, partition, user` |
| `slurm_node_jobs` | running jobs per node per `user` |
| `slurm_collector_ok` | 1 = `scontrol`/`squeue` worked |

Only GPU types Slurm schedules are kept (display cards are dropped). If a source command fails, the file holds only
`*_collector_ok 0` and no numbers; a collector that stops entirely is caught by `CollectorStale`.

## Named queries (recording rules)

Apps should query these names, not their own PromQL, so a measurement changes in one place.

| Name | Meaning |
|---|---|
| `node:cpu_used:ratio` | CPU actually used, 0–1 (5-minute average) |
| `node:mem_used:ratio` | RAM actually used |
| `node:disk_used:ratio` | root filesystem (`/`) used |
| `node:load1_per_cpu:ratio` | load per core |
| `gpu:util:ratio` | GPU busy |
| `gpu:mem_used:ratio`, `gpu:mem_used:bytes` | GPU memory used |
| `gpu:temp:celsius` | GPU temperature |
| `slurm:node_cpus_alloc:ratio` | CPUs reserved by Slurm |
| `slurm:node_mem_alloc:ratio` | memory reserved by Slurm |
| `slurm:node_gpus_free` | GPUs a new job could get now (0 if the node is down or drained) |
| `slurm:gpus_free` | free GPUs, whole cluster |

**Reserved** = handed to jobs by Slurm. **Used** = what the hardware does. A job can reserve 8 CPUs and use 1.

```bash
curl -s "$PROM_URL/api/v1/query" --data-urlencode 'query=slurm:node_gpus_free'
```

## Alerts

No email or paging yet: alerts show in the Cluster monitor and at `$PROM_URL/alerts`.

### Platform alerts (`rules.yml`)

| Alert | Fires when | For | Severity | What to do |
|---|---|---|---|---|
| `TargetDown` | `up == 0` | 5m | critical | `ping`; `systemctl status node_exporter` on the node; rerun `1-install.sh` when it's back |
| `ServiceDown` | a `MONITOR_UNITS` unit isn't active | 2m | critical | `systemctl status/start <unit>`, `journalctl -u <unit> -n 50`. LDAP or sssd down = nobody can log in |
| `ServiceUnreachable` | a probe fails | 3m | critical | open the URL from master; check httpd, keycloak, `kubectl get pods -n mlflow` / `-n model-hub`, the registry, `dirsrv@*` |
| `CertificateExpiresSoon` | 7–30 days left | 1h | warning | plan the renewal (see [known issues](../known-issues/install-and-docs.md#certificate-renewal-isnt-really-documented)) |
| `CertificateExpiresVerySoon` | < 7 days left | 10m | critical | renew now |
| `SlurmNodeUnavailable` | node down/drained/not responding | 10m | warning | `scontrol show node <n>`; fix; `scontrol update nodename=<n> state=resume` |
| `CollectorFailing` | `gpu_collector_ok` or `slurm_collector_ok` is 0 | 5m | warning | `journalctl -u gpu-metrics` / `-u slurm-metrics`; `nvidia-smi` |
| `CollectorStale` | a collector file is older than 120 s | 5m | warning | `systemctl list-timers gpu-metrics.timer slurm-metrics.timer` |
| `GpuEccErrors` | uncorrectable ECC errors > 0 | 1m | critical | drain, wait for jobs, `nvidia-smi -r -i <gpu>`; replace the card if it recurs |
| `GpuRemapPending` | a remap waits for a reset | 5m | warning | when idle: drain, reset, resume |
| `GpuThrottled` | hardware slowdown | 10m | warning | `nvidia-smi -q -d PERFORMANCE,TEMPERATURE,POWER`; check airflow and power |
| `GpuMissing` | Slurm has more GPUs than `nvidia-smi` sees | 10m | critical | `nvidia-smi`; `dmesg \| grep -i xid`; drain, reboot, reseat or replace |
| `GpuHot` | GPU above 85 °C | 10m | warning | fans, room; `nvidia-smi -q -d TEMPERATURE` |
| `PrometheusBadConfig` | a reload was rejected | 10m | critical | `promtool check config …`; fix and rerun `1-install.sh` |
| `PrometheusRuleFailures` | a rule errors on every evaluation | 15m | critical | `journalctl -u prometheus \| grep 'Evaluating rule failed'` |
| `PrometheusTSDBCompactionsFailing` | data blocks can't be merged | 4h | warning | `df -h /var/lib/prometheus`; journal |

### Node alerts (`node-alerts.yml`, node-mixin 1.12.1)

Filesystems are limited to local `ext4|xfs|btrfs`.

| Alert | Fires when | Severity | What to do |
|---|---|---|---|
| `NodeFilesystemSpaceFillingUp` | < 40 % free and full within 24 h / < 20 % and within 4 h | warning / critical | `du -xh --max-depth=2 <mount> \| sort -h \| tail` |
| `NodeFilesystemAlmostOutOfSpace` | < 5 % / < 3 % free | warning / critical | same |
| `NodeFilesystemFilesFillingUp` | inodes running out within 24 h / 4 h | warning / critical | find folders with many files |
| `NodeFilesystemAlmostOutOfFiles` | < 5 % / < 3 % inodes free | warning / critical | same |
| `NodeNetworkReceiveErrs`, `NodeNetworkTransmitErrs` | > 1 % errors | warning | `ip -s link show <dev>`; cable, port, NIC |
| `NodeHighNumberConntrackEntriesUsed` | conntrack > 75 % | warning | raise `nf_conntrack_max` or find the source |
| `NodeTextFileCollectorScrapeError` | node_exporter can't read a `.prom` file | warning | `ls -l /var/lib/node_exporter/textfile/` |
| `NodeClockSkewDetected`, `NodeClockNotSynchronising` | clock off / NTP not syncing | warning | `chronyc tracking`, `chronyc sources`. Munge and TLS break with bad clocks |
| `NodeRAIDDegraded` | RAID lost a disk | critical | `cat /proc/mdstat`; `mdadm --detail`; replace now |
| `NodeRAIDDiskFailure` | a RAID member failed | warning | same |
| `NodeFileDescriptorLimit` | fd table > 70 % / > 90 % | warning / critical | `cat /proc/sys/fs/file-nr` |
| `NodeMemoryMajorPagesFaults` | heavy paging | warning | `vmstat 1`; `squeue -w <node>` |
| `NodeMemoryHighUtilization` | RAM > 90 % | warning | `squeue -w <node>`; a job without `--mem`? |
| `NodeDiskIOSaturation` | disk queue high | warning | `iostat -x 1`; NFS load on master |
| `NodeSystemdServiceCrashlooping` | a unit keeps restarting | warning | `systemctl status`, `journalctl -u <unit> -n 100` |
| `NodeBondingDegraded` | a bond lost a member | warning | `cat /proc/net/bonding/<bond>` |

Dropped on purpose: `NodeCPUHighUsage` and `NodeSystemSaturation` (100 % CPU is the point of HPC) and
`NodeSystemdServiceFailed` (`ServiceDown` covers it).
