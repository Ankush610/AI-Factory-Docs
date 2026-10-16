# Monitoring reference

Metrics, named queries and alerts provided by [9-monitoring](../admin/install/monitoring.md). What to do when an
alert fires is in [Alerts](#alerts): the Cluster monitor's **How to fix** links open its rows.

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

<a id="alerts-what-to-do"></a>

## Alerts

Every alert, what it means and what to do. This is the runbook: the Cluster monitor's **How to fix** links open
`RUNBOOK_URL#<alert name in lower case>` (each row has that anchor), and **Every alert and how to fix it** opens
`#alerts-what-to-do`. Keep the anchors when editing.

No email or paging yet: alerts show in the Cluster monitor and at `$PROM_URL/alerts`.

### Platform alerts (`rules.yml`)

| Alert | Fires when | For | Severity | What to do |
|---|---|---|---|---|
| <a id="targetdown"></a>`TargetDown` | `up == 0`: a node is off or unreachable, or its node_exporter (or blackbox_exporter) stopped | 5m | critical | `ping <node>`; `ssh <node> systemctl status node_exporter`. When it's back, rerun `1-install.sh` (it was skipped). Planned downtime: drain it in Slurm |
| <a id="servicedown"></a>`ServiceDown` | a `MONITOR_UNITS` unit stopped or failed | 2m | critical | `ssh <node> systemctl status <unit>`; `journalctl -u <unit> -n 50`; `systemctl start <unit>`. LDAP (`dirsrv@*`) or sssd down = nobody can log in |
| <a id="serviceunreachable"></a>`ServiceUnreachable` | a probe fails: the service doesn't answer the way a user calls it, even if systemd says active | 3m | critical | Open the URL in the alert from master. Portal/Keycloak: `systemctl status httpd keycloak`. MLflow/gateway: `kubectl get pods -n mlflow` / `-n model-hub`. Registry: `curl http://master:5000/v2/`, `podman ps`. LDAP: `systemctl status dirsrv@*` |
| <a id="certificateexpiressoon"></a>`CertificateExpiresSoon` | a certificate (portal, LDAP, gateway) has 7–30 days left | 1h | warning | Plan the renewal. Rerunning the install scripts doesn't renew every certificate: see [known issues](../known-issues/install-and-docs.md#certificate-renewal-isnt-really-documented). Customer certificates: ask them for a new one |
| <a id="certificateexpiresverysoon"></a>`CertificateExpiresVerySoon` | < 7 days left | 10m | critical | Renew now (as above). After it expires, browsers and `ldaps` clients refuse to connect |
| <a id="slurmnodeunavailable"></a>`SlurmNodeUnavailable` | a Slurm node is down, drained or not responding: it takes no jobs | 10m | warning | `scontrol show node <node>` (see `Reason=`); fix it, then `scontrol update nodename=<node> state=resume` |
| <a id="collectorfailing"></a>`CollectorFailing` | `gpu_collector_ok` or `slurm_collector_ok` is 0: nvidia-smi or scontrol/squeue fails | 5m | warning | `journalctl -u gpu-metrics -n 20` (GPU nodes) or `-u slurm-metrics` (master); `ssh <node> nvidia-smi` |
| <a id="collectorstale"></a>`CollectorStale` | a collector file is older than 120 s (timer off or killed) | 5m | warning | `systemctl list-timers gpu-metrics.timer slurm-metrics.timer`; `systemctl start <name>.timer` |
| <a id="gpueccerrors"></a>`GpuEccErrors` | uncorrectable ECC errors > 0: jobs on the GPU can crash or compute garbage | 1m | critical | Drain the node (`scontrol update nodename=<node> state=drain reason=ecc`), wait for jobs, `nvidia-smi -r -i <gpu>`. Again after a reset = replace the card |
| <a id="gpuremappending"></a>`GpuRemapPending` | a bad memory row is replaced at the next GPU reset | 5m | warning | When the GPU is idle: drain, `nvidia-smi -r -i <gpu>`, resume. Not urgent |
| <a id="gputhrottled"></a>`GpuThrottled` | hardware slowed the GPU down (power brake or heat) | 10m | warning | `nvidia-smi -q -d PERFORMANCE,TEMPERATURE,POWER` on the node; check airflow, PSU, power cables |
| <a id="gpumissing"></a>`GpuMissing` | Slurm has more GPUs on a node than `nvidia-smi` sees: a card fell off the bus | 10m | critical | `nvidia-smi`; `dmesg \| grep -i xid`; drain and reboot the node; still missing = reseat or replace the card |
| <a id="gpuhot"></a>`GpuHot` | GPU above 85 °C | 10m | warning | Check fans and room cooling; `nvidia-smi -q -d TEMPERATURE` |
| <a id="prometheusbadconfig"></a>`PrometheusBadConfig` | a reload was rejected: Prometheus still runs the old config | 10m | critical | `promtool check config /etc/prometheus/prometheus.yml`; `journalctl -u prometheus -n 50`; fix and rerun `1-install.sh` |
| <a id="prometheusrulefailures"></a>`PrometheusRuleFailures` | a rule errors on every evaluation: its alert can't fire | 15m | critical | `journalctl -u prometheus \| grep 'Evaluating rule failed'` shows the rule and error; fix it in `rules.yml`, add a case to `rules_test.yml` |
| <a id="prometheustsdbcompactionsfailing"></a>`PrometheusTSDBCompactionsFailing` | data blocks can't be merged | 4h | warning | Disk space under `PROM_DATA` (`df -h /var/lib/prometheus`); `journalctl -u prometheus -n 50` |

### Node alerts (`node-alerts.yml`, node-mixin 1.12.1)

Filesystems are limited to local `ext4|xfs|btrfs`.

| Alert | Fires when | Severity | What to do |
|---|---|---|---|
| <a id="nodefilesystemspacefillingup"></a>`NodeFilesystemSpaceFillingUp` | < 40 % free and full within 24 h / < 20 % and within 4 h | warning / critical | `du -xh --max-depth=2 <mount> \| sort -h \| tail`; on `/home` look for big user folders, old SIFs, `/home/apps/registry` |
| <a id="nodefilesystemalmostoutofspace"></a>`NodeFilesystemAlmostOutOfSpace` | < 5 % / < 3 % free | warning / critical | Same as above, now |
| <a id="nodefilesystemfilesfillingup"></a>`NodeFilesystemFilesFillingUp` | inodes (number of files) running out within 24 h / 4 h | warning / critical | `find <mount> -xdev -type f \| cut -d/ -f1-4 \| sort \| uniq -c \| sort -n \| tail` (millions of small files: datasets, caches) |
| <a id="nodefilesystemalmostoutoffiles"></a>`NodeFilesystemAlmostOutOfFiles` | < 5 % / < 3 % inodes free | warning / critical | Same as above, now |
| <a id="nodenetworkreceiveerrs"></a>`NodeNetworkReceiveErrs` | > 1 % receive errors on an interface | warning | `ip -s link show <dev>`; cable, switch port, NIC |
| <a id="nodenetworktransmiterrs"></a>`NodeNetworkTransmitErrs` | > 1 % transmit errors on an interface | warning | Same as above |
| <a id="nodehighnumberconntrackentriesused"></a>`NodeHighNumberConntrackEntriesUsed` | connection-tracking table > 75 % full (k8s nodes) | warning | `sysctl net.netfilter.nf_conntrack_max`; raise it, or find who opens so many connections |
| <a id="nodetextfilecollectorscrapeerror"></a>`NodeTextFileCollectorScrapeError` | node_exporter can't read a `.prom` file | warning | `ls -l /var/lib/node_exporter/textfile/`; `curl -s localhost:9100/metrics \| grep textfile` |
| <a id="nodeclockskewdetected"></a>`NodeClockSkewDetected` | clock off by more than 0.05 s | warning | `chronyc tracking`; `systemctl status chronyd`. Munge (Slurm) and TLS break with bad clocks |
| <a id="nodeclocknotsynchronising"></a>`NodeClockNotSynchronising` | clock not syncing at all | warning | `chronyc sources`; is the NTP server reachable? |
| <a id="noderaiddegraded"></a>`NodeRAIDDegraded` | a RAID array lost a disk | critical | `cat /proc/mdstat`; `mdadm --detail /dev/<device>`; replace the disk now: one more failure = data loss |
| <a id="noderaiddiskfailure"></a>`NodeRAIDDiskFailure` | a disk in a RAID array failed | warning | Same as above |
| <a id="nodefiledescriptorlimit"></a>`NodeFileDescriptorLimit` | the kernel's open-file table > 70 % / > 90 % used | warning / critical | `cat /proc/sys/fs/file-nr`; `lsof \| awk '{print $1}' \| sort \| uniq -c \| sort -n \| tail` |
| <a id="nodememorymajorpagesfaults"></a>`NodeMemoryMajorPagesFaults` | heavy swapping / page-ins: the node crawls | warning | `vmstat 1`; which job: `squeue -w <node>`, `top -o RES` |
| <a id="nodememoryhighutilization"></a>`NodeMemoryHighUtilization` | RAM > 90 % for 15 min | warning | `squeue -w <node>`, `top -o RES`; a job without a memory limit (Slurm `--mem`)? |
| <a id="nodediskiosaturation"></a>`NodeDiskIOSaturation` | disk I/O queue high for 30 min | warning | `iostat -x 1`; on master: NFS load from many jobs at once |
| <a id="nodesystemdservicecrashlooping"></a>`NodeSystemdServiceCrashlooping` | a systemd unit keeps restarting | warning | `systemctl status <unit>`; `journalctl -u <unit> -n 100` |
| <a id="nodebondingdegraded"></a>`NodeBondingDegraded` | a bonded network link lost a member | warning | `cat /proc/net/bonding/<bond>`; cable, switch port |

Dropped on purpose: `NodeCPUHighUsage` and `NodeSystemSaturation` (100 % CPU is the point of HPC) and
`NodeSystemdServiceFailed` (`ServiceDown` covers it).
