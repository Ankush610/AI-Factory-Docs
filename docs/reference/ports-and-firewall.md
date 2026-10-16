# Ports and firewall

Every network port the platform uses, which host it is on, and who may reach it.

## All ports

| Port | Host (bind address) | Service | Firewall |
|---|---|---|---|
| 22 | master | SSH: master → nodes as root, and the Admin page's restricted key from the login node | not opened by the scripts (prerequisite) |
| 443 | login node | Portal (Apache + Open OnDemand); also proxies `/auth` to Keycloak | `https` service, **all sources** (`2-ood`) |
| 389 | master | LDAP, plain. Only master's own scripts use it (`ldap://localhost`) | `ldap` service, **all sources** (`1-ldap`) |
| 636 | master | LDAPS: what every node and Keycloak use | `ldaps` service, **all sources** (`1-ldap`) |
| 5000 | master | Container registry, plain HTTP | not opened by the scripts; open it to `LOGIN_IP` and the nodes |
| 6443 | master | Kubernetes API (users' `kubectl` from the login node) | prerequisite |
| 6817–6819 | master | Slurm (`slurmctld`, `slurmd`, `slurmdbd`) | prerequisite |
| 8180 (`KEYCLOAK_PORT`) | `MASTER_IP` | Keycloak, plain HTTP | rich rule: **`LOGIN_IP` only** (`6-keycloak`) |
| 9180 (`KEYCLOAK_MGMT_PORT`) | 127.0.0.1 | Keycloak health and metrics | not reachable from outside |
| 5433 (`KEYCLOAK_DB_PORT`) | 127.0.0.1 | Keycloak's Postgres | not reachable from outside |
| 30500 (`MLFLOW_PORT`) | `MASTER_IP` (Kubernetes NodePort) | MLflow | not opened by the scripts; open it to `LOGIN_IP` and the compute nodes |
| 9090 (`PROM_PORT`) | `MASTER_IP` | Prometheus | rich rule: **`LOGIN_IP` only** (and master) (`9-monitoring`) |
| 9115 (`BLACKBOX_PORT`) | 127.0.0.1 | blackbox_exporter | not reachable from outside |
| 9100 (`NODE_EXPORTER_PORT`) | every node | node_exporter | rich rule: **`MASTER_IP` only** (`9-monitoring`) |
| 443 and 8444 | `GATEWAY_IP` (MetalLB) | Model Hub gateway. 8444 = "Shared with me" (needs the user's cluster certificate) | Kubernetes Service; no firewalld rule |
| 8080 | an endpoint's ClusterIP | One Model Hub endpoint, reached with `ssh -L 8080:<ClusterIP>:8080` | inside the cluster only |
| 20000–59999 (random) | compute node or the viewer node | one VS Code or JupyterLab session, reached through the portal's `/rnode` or `/node` proxy | not opened |

Users outside the rack need only **two addresses**: the portal (443) and the gateway (443).

## Rules the scripts add

The scripts add a firewalld rule only when firewalld is running. Where a rule is missing, the Keycloak, Prometheus
and node_exporter steps print `WARNING: firewalld isn't running … open to the whole network`. `9-monitoring` never
turns firewalld on.

| Host | Rule | Added by |
|---|---|---|
| master | services `ldap` (389) and `ldaps` (636), all sources | `1-ldap/1-server.sh` |
| login node | service `https` (443), all sources | `2-ood/setup-ood.sh` |
| master | `KEYCLOAK_PORT`/tcp from `LOGIN_IP` only | `6-keycloak/1-install.sh` |
| master | `PROM_PORT`/tcp from `LOGIN_IP` only | `9-monitoring/1-install.sh` |
| every node | `NODE_EXPORTER_PORT`/tcp from `MASTER_IP` only | `9-monitoring/node-setup.sh` |

## Ports you must open yourself

If the login node is a separate machine, open these **on master** for `LOGIN_IP`:

- 22 (Admin page helper), 636 (LDAPS), `KEYCLOAK_PORT` (`6-keycloak` adds this one),
- 6443 (Kubernetes API), 30500 (MLflow), 6817–6819 (Slurm), 5000 (registry, for podman pushes).

Port 443 must be open on the login node.

!!! warning "Open by default"
    The registry (5000) has no login and allows deletes, and LDAP 389/636 are open to every source. See
    [Known issues → Security](../known-issues/security.md).
