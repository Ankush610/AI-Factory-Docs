# `site.conf` reference

`OOD-Install/site.conf` is the **only file you edit** for a new cluster. Every install script sources it, and no
script hardcodes a hostname, IP, base DN, port or path.

!!! warning "Set every value before the first run"
    The LDAP base DN and every user ID are written into data during install. You can't change them later without
    rebuilding the directory.

- The file is plain bash. Later values reuse earlier ones (`LOGIN_HOST=$MASTER_HOST`, `REGISTRY=$MASTER_HOST:5000`),
  so changing `MASTER_HOST` or `MASTER_IP` updates the values built from them.
- **Example** shows the test cluster's value. Site-specific addresses are written as placeholders, such as `<master-ip>`.
- **Used by** lists the install folders (`0` = `0-registry` … `9` = `9-monitoring`) whose scripts or templates read the variable.
- After you change a value, rerun the scripts of the folders that use it. If a portal app is affected, click
  **Restart Web Server** in the portal afterwards.

## Nodes

| Variable | Example | Meaning | Used by |
|---|---|---|---|
| `MASTER_HOST` | `master` | Master's short hostname. Must equal `hostname -s` on master and resolve to `MASTER_IP` on every node | 0–6, 8, 9 |
| `MASTER_IP` | `<master-ip>` | Master's IP: MLflow NodePort, Keycloak, Prometheus; the only source allowed to node_exporter | 3, 6, 8, 9 |
| `LOGIN_HOST` | `$MASTER_HOST` | Login node: the portal, users' SSH, the `viewer` partition. Must equal its `hostname -s` | 1–9 |
| `LOGIN_IP` | `$MASTER_IP` | The login node's IP as master sees it. The only source let in to Keycloak, Prometheus and the admin page's ssh key | 6, 8, 9 |
| `COMPUTE_NODES` | `cn01 cn02` | Every node that runs jobs | 0, 1, 4, 7, 9 |
| `LOGIN_NODES` | `$LOGIN_HOST` | Nodes where users SSH in and build images with rootless podman (they get subuids there) | 0, 1, 9 |

## Container registry (`0-registry`)

| Variable | Example | Meaning | Used by |
|---|---|---|---|
| `REGISTRY` | `$MASTER_HOST:5000` | Registry address. podman pushes here and containerd pulls from here (plain HTTP, cluster LAN) | 0, 3, 5, 6, 9 |
| `REGISTRY_DATA` | `/home/apps/registry` | Registry storage (the image layers). Back this up | 0 |
| `REGISTRY_IMAGE` | `docker.io/library/registry:2` | The registry's own container image | 0 |

## LDAP (`1-ldap`)

| Variable | Example | Meaning | Used by |
|---|---|---|---|
| `LDAP_BASE` | `dc=aistack,dc=local` | Root of the user tree. **Can't change after install** | 1, 2, 3, 5, 6, 8, 9 |
| `LDAP_INSTANCE` | `aistack` | 389 DS instance name (`/etc/dirsrv/slapd-aistack`) | 1 |
| `LDAP_URI` | `ldaps://$MASTER_HOST` | What the nodes use (port 636, TLS). The name must match the certificate | 1, 2, 3, 5, 6, 8, 9 |
| `LDAP_DM` | `cn=Directory Manager` | The LDAP super-admin | 1, 6 |
| `LDAP_DM_PASS_FILE` | `/root/.ldap-dm.pass` | Its password, written by `1-server.sh`. Root only, no trailing newline | 1, 6 |
| `LDAP_CA` | `/home/apps/ldap/ca.crt` | The LDAP server's CA on the shared `/home`, so every node can trust it | 1, 2, 3, 5, 6, 8, 9 |
| `MIN_UID` | `1000` | First user ID for people. Service accounts stay below it | 1, 3, 5, 6 |
| `MAX_UID` | `59999` | Last user ID for people | 1 |
| `LDAP_LAST_UID_FILE` | `/root/.ldap-last-uid` | Highest user ID ever handed out, so a removed person's ID is never reused. Back it up with LDAP | 1 |
| `KEEP_LOCAL` | `admin` | Local accounts that never go into LDAP: the way back in if LDAP breaks | 1, 8 |

## Open OnDemand (`2-ood`)

| Variable | Example | Meaning | Used by |
|---|---|---|---|
| `OOD_VERSION` | `4.2` | Open OnDemand release | 2 |
| `OOD_SERVERNAME` | `<portal>` | The exact host name users type in the browser | 2, 6, 7 |
| `OOD_CERT` | `/etc/pki/tls/certs/ood.crt` | On the login node: the portal's HTTPS certificate (self-signed by `2-ood`). Keycloak fetches it to trust logouts | 2, 6 |
| `OOD_OIDC_SECRET` | `/etc/ood/oidc-client.secret` | On the login node (root, 0600): Keycloak client `ood`'s secret, pushed by `6-keycloak/2-realm.sh` | 2, 6 |
| `OOD_ADMIN` | `admin` | Local web login (htpasswd) that works even when LDAP is down. The only user of the Admin page and Cluster monitor | 1, 2, 6, 8, 9 |
| `OOD_AUTH` | `keycloak` | Portal login: `ldap` (the browser asks) or `keycloak` (one login; needs `6-keycloak`). Rerun `2-ood/setup-ood.sh` after a change. See the [first-install warning](../admin/index.md#install-order) | 2, 3, 5 |
| `CLUSTER_ID` | `aistack` | Portal cluster file name and ID; also the Keycloak realm | 2, 5, 9 |
| `CLUSTER_TITLE` | `AI-Stack` | Name in the portal's Clusters menu | 2, 5 |
| `PORTAL_TITLE` | `AI-HPC-Cluster` | Portal name: top bar, browser tab, Keycloak login heading | 2, 6 |
| `PORTAL_COLOR` | `#05295d` | Top bar color (navy) | 2 |
| `SLURM_BIN` | `/usr/local/bin` | Folder that holds `sbatch` (`which sbatch`) | 1, 2, 4, 5, 7, 8, 9 |
| `SLURM_CONF` | `/etc/slurm/slurm.conf` | Slurm's config file | 1, 2, 4, 5, 7, 8, 9 |

## Admin page (`8-admin`)

| Variable | Example | Meaning | Used by |
|---|---|---|---|
| `ADMIN_HELPER` | `/usr/local/sbin/aistack-admin` | On master: root-owned copy of `8-admin/aistack-admin.py`. The Admin page reaches it over ssh with `ADMIN_KEY` | 8 |
| `ADMIN_KEY` | `/etc/aistack-admin/id_ed25519` | On the login node, readable by `OOD_ADMIN` only. Master's root accepts it from `LOGIN_IP` and runs only `ADMIN_HELPER` with it | 8 |
| `ADMIN_LOG` | `/var/log/aistack-admin.log` | On master, root only: every change made through the helper (who, what, when, result) | 8 |
| `ADMIN_NOTICES` | `/home/apps/admin-notices` | Sticky folder (mode 1777) on the shared `/home`: one file per job or endpoint the admin stopped, owned by its user. Shown by the portal's bell and the VS Code, Jupyter and Model Hub pages | 2, 4, 7, 8 |

## Keycloak (`6-keycloak`)

| Variable | Example | Meaning | Used by |
|---|---|---|---|
| `KEYCLOAK_VERSION` | `26.8.0` | `quay.io/keycloak/keycloak` tag. Never `latest` | 6 |
| `KEYCLOAK_IMAGE` | `$REGISTRY/keycloak:$KEYCLOAK_VERSION-rudra` | Image built by `1-install.sh` (optimized for Postgres) | 6 |
| `KEYCLOAK_DB_IMAGE` | `$REGISTRY/postgres:17.11` | Postgres image, mirrored from `docker.io/library/postgres` | 6 |
| `KEYCLOAK_USER` | `keycloak` | Local system account that runs both containers and owns `KEYCLOAK_DATA` | 6 |
| `KEYCLOAK_UID` | `949` | Its user ID: below `MIN_UID`, never in LDAP | 6 |
| `KEYCLOAK_DATA` | `/home/apps/keycloak` | Postgres data and admin passwords (master's local disk, mode 700) | 1, 6 |
| `KEYCLOAK_PATH` | `/auth` | URL path under the portal's address | 2, 6 |
| `KEYCLOAK_PORT` | `8180` | Keycloak's HTTP port on `MASTER_IP`. Apache proxies `KEYCLOAK_PATH` here | 6 |
| `KEYCLOAK_URL` | `http://$MASTER_IP:$KEYCLOAK_PORT$KEYCLOAK_PATH` | What the portal's Apache and the scripts call (plain HTTP on the cluster LAN) | 1, 2, 6, 9 |
| `KEYCLOAK_MGMT_PORT` | `9180` | Health and metrics, 127.0.0.1 only | 6 |
| `KEYCLOAK_DB_PORT` | `5433` | Postgres, 127.0.0.1 only (5432 may already be taken on master) | 6 |
| `KEYCLOAK_REALM` | `$CLUSTER_ID` | The realm users log in to | 1, 2, 6, 9 |
| `KEYCLOAK_ADMIN` | `kcadmin` | Keycloak's own admin (master realm). Password in `$KEYCLOAK_DATA/admin.pass` | 1, 6 |

## MLflow (`3-mlflow`)

| Variable | Example | Meaning | Used by |
|---|---|---|---|
| `MLFLOW_VERSION` | `3.15.1` | Must equal `mlflow==` in `3-mlflow/image/requirements.txt` | 3, 5 |
| `MLFLOW_IMAGE` | `$REGISTRY/mlflow-server:$MLFLOW_VERSION-sso` | Server image. `-sso`: the login plugin also trusts the portal's Keycloak login | 3 |
| `MLFLOW_PORT` | `30500` | NodePort on `MASTER_IP` | 3, 5 |
| `MLFLOW_PREFIX` | `/node/$MASTER_IP/$MLFLOW_PORT` | URL path MLflow serves under. `""` for none | 2, 3, 5 |
| `MLFLOW_URI` | `http://$MASTER_IP:$MLFLOW_PORT$MLFLOW_PREFIX` | The tracking URI users and jobs use | 1, 3, 5, 9 |
| `MLFLOW_DATA` | `/home/apps/mlflow-shared` | Runs, model registry, artifacts and MLflow users (master's disk) | 1, 3, 5 |
| `MLFLOW_USER` | `mlflow` | Local system account that owns `MLFLOW_DATA` and runs the pod | 3, 5 |
| `MLFLOW_UID` | `950` | Its user ID: below `MIN_UID` | 3 |
| `MLFLOW_PROXY_SECRET` | `/etc/ood/mlflow-proxy.secret` | On the login node (root, 0600): copy of `MLFLOW_DATA/proxy.secret` for the portal's Apache, pushed by `2-deploy.sh` | 2, 3 |

## VS Code (`4-vscode`)

| Variable | Example | Meaning | Used by |
|---|---|---|---|
| `CODE_SERVER_VERSION` | `4.139.1` | code-server release (github.com/coder/code-server) | 4 |
| `CODE_SERVER_ROOT` | `/home/apps/code-server` | One install on the shared `/home`; every node runs it | 4 |
| `VIEWER_PARTITION` | `viewer` | Slurm partition for "Editor only" and "Notebook only" sessions, on the login node | 4, 7, 9 |
| `VIEWER_NODE` | `viewer01` | The login node's **Slurm** name. Not its hostname: slurm-bridge taints Kubernetes nodes whose name matches a Slurm node | 4 |
| `VIEWER_RESERVED_CORES` | `8` | Slurm `CoreSpecCount`: cores kept out of Slurm for SSH users and services | 4 |
| `VIEWER_RESERVED_MEM_MB` | `49152` | Slurm `MemSpecLimit` (48 GB) | 4 |
| `VIEWER_OVERSUBSCRIBE` | `4` | Tool sessions per core (an idle editor uses almost no CPU) | 4 |
| `VSCODE_MEM` | `2G` | Memory per editor session | 4 |
| `VSCODE_IDLE_SECONDS` | `3600` | An editor with no browser attached for this long stops | 4 |
| `GPU_PARTITION` | `auto` | `auto` = Slurm's default partition with GPUs, else the first one with GPUs; or a partition name; `""` = no GPU choice | 4, 7 |

## Model Hub (`5-model-hub`)

| Variable | Example | Meaning | Used by |
|---|---|---|---|
| `MODELS_ROOT` | `/home/apps/models` | Admin base models (`base/`) and public copies (`public/`). Read-only to users and pods | 5 |
| `MODELHUB_USER` | `modelhub` | Local system account that owns `MODELS_ROOT` | 5 |
| `MODELHUB_UID` | `951` | Its user ID: below `MIN_UID` | 5 |
| `BRIDGE_PARTITION` | `slurm-bridge` | Slurm partition slurm-bridge puts pods in | 5, 8 |
| `BRIDGE_NS` | `slurm` | Namespace of the slurm-bridge Helm release | 5 |
| `USER_GPU_QUOTA` | `1` | GPUs one user's endpoints may hold at once | 5, 8 |
| `ENDPOINT_HOURS_DEFAULT` | `24` | Endpoint run time when the user doesn't choose | 5 |
| `ENDPOINT_HOURS_MAX` | `96` | Longest an endpoint may run. Enforced by the cluster, not only the page | 5, 8 |
| `NFS_SERVER` | `$MASTER_IP` | Serves `/home` to pods (per-user volumes) | 5 |
| `APPTAINER` | `/home/apps/apptainer/bin/apptainer` | The Apptainer binary | 5, 7 |
| `CONTAINERS_ROOT` | `/home/apps/containers` | Apptainer images: `training/<family>/<name>.sif`, `serving/`, `test-scripts/` | 5, 7 |
| `ML_TRAIN_SIF` | `$CONTAINERS_ROOT/training/ml-classic/ml-classic-cpu.sif` | Classic-ML training image. `model-register` runs in it; rerun `5-model-hub/1-setup.sh` after a change | 5 |
| `TORCH_TRAIN_SIF` | `$CONTAINERS_ROOT/training/pytorch/pytorch-2.10.0-cuda12.9.sif` | PyTorch training image the serving image is built to match | 5 |
| `VLLM_VERSION` | `v0.30.0` | `docker.io/vllm/vllm-openai` tag mirrored to the registry. Never `latest` | 5 |
| `GATEWAY_NS` | `model-hub` | Gateway's Kubernetes namespace (not Slurm-managed, pinned to master) | 5 |
| `GATEWAY_IP` | `<gateway-ip>` | Fixed address from the MetalLB pool; on a customer rack, an IP on their network | 5 |
| `GATEWAY_HOST` | `""` | DNS name for the gateway, if the site has one. Goes into the certificate | 5 |
| `GATEWAY_URL` | `https://${GATEWAY_HOST:-$GATEWAY_IP}` | What users call | 5, 9 |
| `GATEWAY_TLS_SECRET` | `""` | `""` = a self-signed Rudra CA (users download `ca.crt`); or a Kubernetes TLS Secret the customer provides | 5 |
| `GATEWAY_CA` | `/home/apps/gateway/ca.crt` | That self-signed CA (public), on the shared `/home`. Model Hub offers it for download | 5 |
| `GATEWAY_NGINX_IMAGE` | `$REGISTRY/nginx-unprivileged:1.30.5-alpine` | Mirrored from `docker.io/nginxinc` | 5 |
| `GATEWAY_PY_IMAGE` | `$REGISTRY/python:3.12.15-slim` | Mirrored from `docker.io/library`; runs the gateway's key checker | 5 |
| `SSH_JUMP` | `<jump-host>` | Jump host shown in Model Hub's `ssh -L 8080:…` hint (`host` or `user@host`; `""` = none) | 5 |

## Monitoring (`9-monitoring`)

| Variable | Example | Meaning | Used by |
|---|---|---|---|
| `PROMETHEUS_VERSION` | `3.5.5` | Prometheus release (3.5 is long-term support). Never `latest` | 9 |
| `PROMETHEUS_SHA256` | `64d0beab…` | sha256 of the release tarball. Change it with the version | 9 |
| `NODE_EXPORTER_VERSION` | `1.12.1` | node_exporter release | 9 |
| `NODE_EXPORTER_SHA256` | `b51d8a76…` | Its tarball's sha256 | 9 |
| `BLACKBOX_VERSION` | `0.28.0` | blackbox_exporter release (tries each service the way a user would) | 9 |
| `BLACKBOX_SHA256` | `caf5d242…` | Its tarball's sha256 | 9 |
| `MONITOR_ROOT` | `/home/apps/monitoring` | Downloaded releases on the shared `/home`; nodes install from here with no internet | 9 |
| `PROM_DATA` | `/var/lib/prometheus` | Prometheus' database on master's **local** disk (it must not be on NFS) | 9 |
| `PROM_PORT` | `9090` | Prometheus on `MASTER_IP`. firewalld lets in only `LOGIN_IP` and master | 9 |
| `PROM_URL` | `http://$MASTER_IP:$PROM_PORT` | What the portal apps call (read-only `/api/v1/query`) | 9 |
| `PROM_RETENTION` | `30d` | History kept this long… | 9 |
| `PROM_RETENTION_SIZE` | `20GB` | …or until it reaches this size, whichever comes first | 9 |
| `RUNBOOK_URL` | this site's [Monitoring reference](monitoring.md#alerts) | "How to fix" links go to `RUNBOOK_URL#<alertname in lower case>` | 9 |
| `BLACKBOX_PORT` | `9115` | blackbox_exporter, 127.0.0.1 on master only | 9 |
| `NODE_EXPORTER_PORT` | `9100` | On every node. firewalld lets in only `MASTER_IP` | 9 |
| `MONITOR_UNITS` | `slurmctld\|slurmd\|…\|nfs-server` | Regex of systemd unit names (no `.service`) watched by the `ServiceDown` alert. Each node watches the ones it has enabled | 9 |

## Jupyter (`7-jupyter`)

| Variable | Example | Meaning | Used by |
|---|---|---|---|
| `JUPYTER_ROOT` | `/home/apps/jupyter` | One install (its own Python + JupyterLab) on the shared `/home` | 7 |
| `JUPYTER_PYTHON` | `3.12.15+20261003` | python-build-standalone build. JupyterLab needs Python ≥ 3.10; AlmaLinux 9 has 3.9 | 7 |
| `JUPYTER_MEM` | `4G` | Memory per "Notebook only" session | 7 |
| `JUPYTER_IDLE_SECONDS` | `3600` | A kernel idle this long with no browser attached stops; then the session stops | 7 |

## Version pins at a glance

| Component | Version | Note |
|---|---|---|
| Open OnDemand | 4.2 | |
| MLflow | 3.15.1 | Must be ≥ the MLflow client in the training images. Database upgrades are one-way |
| Keycloak / Postgres | 26.8.0 / 17.11 | |
| code-server | 4.139.1 | |
| JupyterLab's Python | 3.12.15+20261003 | |
| vLLM | v0.30.0 | |
| nginx-unprivileged / Python (gateway) | 1.30.5-alpine / 3.12.15-slim | |
| Prometheus / node_exporter / blackbox_exporter | 3.5.5 / 1.12.1 / 0.28.0 | sha256-checked |
| Registry | `registry:2` | The one image not pinned to an exact version ([known issue](../known-issues/registry.md)) |
