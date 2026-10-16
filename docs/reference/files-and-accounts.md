# Files, accounts and secrets

Where the platform keeps its data, which system accounts it creates, and where each secret lives.

## Shared storage: `/home/apps`

`/home` is exported from master to every node, so everything under `/home/apps` sits on master's disk and is
mounted over NFS everywhere else. Prometheus' database is deliberately **not** here: it lives on master's local
disk, because it must not be on NFS.

```text
/home/apps/
├── bin/                       shared tools, on every node's PATH (/etc/profile.d/apps-bin.sh)
│   ├── k8s-image              build/push/mirror images (0-registry)
│   └── model-register         register an LLM folder in MLflow (5-model-hub)
├── registry/                  REGISTRY_DATA: the registry's image layers (back up)
├── ldap/ca.crt                LDAP_CA, mode 644
├── admin-notices/             ADMIN_NOTICES, mode 1777: "stopped by the admin" notes, one file per job
├── keycloak/                  KEYCLOAK_DATA, mode 700, owner keycloak (949)
│   ├── db/                    Postgres data
│   ├── themes/
│   └── *.pass                 admin, bootstrap, db, ldap passwords (root 0600)
├── mlflow-shared/             MLFLOW_DATA, mode 700, owner mlflow (950); /data inside the pod
│   ├── mlflow.db              runs and model registry (SQLite)
│   ├── auth.db                MLflow users, admin and job tokens
│   ├── artifacts/             model files and other artifacts
│   ├── basic_auth.ini
│   ├── admin.pass
│   └── proxy.secret
├── code-server/               CODE_SERVER_ROOT: <version>/ and current -> <version>
├── jupyter/                   JUPYTER_ROOT: <lab>-py<python>/ and current -> …
├── models/                    MODELS_ROOT, owner modelhub (951): base/, public/
├── apptainer/bin/apptainer    APPTAINER (installed outside OOD-Install)
├── containers/                CONTAINERS_ROOT (installed outside OOD-Install)
│   ├── apptainer.sh, README.md
│   ├── training/<family>/<name>.sif
│   ├── serving/
│   └── test-scripts/
├── gateway/ca.crt             GATEWAY_CA, mode 644 (public)
└── monitoring/                MONITOR_ROOT: downloaded releases + collectors
```

### Files in each user's home

| Path | Written by | Holds |
|---|---|---|
| `~/.mlflow/credentials` | `3-mlflow/3-sync-tokens.sh` | the user's MLflow job token |
| `~/.kube/aistack.config` | `5-model-hub/2-sync-users.sh` | the user's Kubernetes login (client certificate, 1 year) |
| `~/.local/share/code-server/` | code-server | VS Code extensions and settings |
| `~/.vscode-sessions/`, `~/.jupyter-sessions/` | the session job | connect details (mode 600) and logs per session |
| `~/models/` | the user | their own model files (for example, fine-tuned LLMs) |
| `~/model-downloads/` | Model Hub | models copied "to my cluster home" |

## Accounts and user IDs

| Account | User ID | Where | Purpose |
|---|---|---|---|
| People | `MIN_UID`–`MAX_UID` (1000–59999), never reused | LDAP | users; one private group each |
| People's subuids/subgids | from 100000, 65536 each | `/etc/subuid`, `/etc/subgid` on `LOGIN_NODES` | rootless podman |
| `keycloak` | 949 | local, master | runs Keycloak and Postgres |
| `mlflow` | 950 | local, master | owns `MLFLOW_DATA`; the MLflow pod runs as it |
| `modelhub` | 951 | local, master | owns `MODELS_ROOT` |
| `prometheus` | system (automatic) | local, master | runs Prometheus |
| `node_exporter` | system (automatic) | local, every node | runs node_exporter and the collectors |
| blackbox_exporter | systemd `DynamicUser` | master | probes the services |
| `admin` (`OOD_ADMIN`, in `KEEP_LOCAL`) | local | every node, never in LDAP | the way back in; the only Admin page user |
| `cn=Directory Manager` | — | LDAP | LDAP super-admin |
| `cn=keycloak-passwords,<LDAP_BASE>` | — | LDAP | Keycloak's LDAP account: may write `userPassword` under `ou=People`, nothing else |
| `kcadmin` | — | Keycloak master realm | Keycloak's own admin |
| gateway pod | 101 | Kubernetes | nginx-unprivileged |
| Model Hub endpoint pods | the owner's ID | Kubernetes, `u-<user>` | pods run as their owner |

Service accounts sit below `MIN_UID`, so they can never clash with a person in LDAP.

## Kubernetes namespaces

| Namespace | What runs there |
|---|---|
| `mlflow` | the MLflow pod. Outside slurm-bridge's managed namespaces, so it has no Slurm time limit |
| `model-hub` (`GATEWAY_NS`) | the gateway |
| `u-<user>` | one per person: their Model Hub endpoints |
| `slurm` (`BRIDGE_NS`) | slurm-bridge |
| `monitoring` | KEDA's own Prometheus. **Not** the platform's Prometheus, which runs under systemd on master |

## Portal apps

Installed under `/var/www/ood/apps/sys/` on the login node.

| App folder | Installed by | Who sees it |
|---|---|---|
| `aistack_notices` | `2-ood/setup-ood.sh` | everyone (backend of the notices bell) |
| `mlflow_k8s` | `3-mlflow/4-install-ood-app.sh` | everyone |
| `vscode` | `4-vscode/3-install-ood-app.sh` | everyone |
| `model_hub` | `5-model-hub/7-install-ood-app.sh` | everyone |
| `jupyter` | `7-jupyter/2-install-ood-app.sh` | everyone |
| `aistack_admin` | `8-admin/2-install-ood-app.sh` | `OOD_ADMIN` only |
| `cluster_status` | `9-monitoring/2-install-ood-app.sh` | `OOD_ADMIN` only |

## Secrets

No password, token or private key is stored in the OOD-Install repository. The install scripts generate them on
the cluster.

| Secret | Host | Path | Permissions | Written by |
|---|---|---|---|---|
| LDAP Directory Manager password | master | `/root/.ldap-dm.pass` | root only, no trailing newline | `1-ldap/1-server.sh` |
| Highest user ID handed out | master | `/root/.ldap-last-uid` | root only | `1-ldap/add-user.sh` |
| SSSD config | every node | `/etc/sssd/sssd.conf` | `root:root 0600` | `1-ldap/2-client.sh` |
| Portal TLS key | login node | `/etc/pki/tls/private/ood.key` | root | `2-ood/setup-ood.sh` |
| Portal admin login (htpasswd) | login node | `/etc/ood/htpasswd` | not set explicitly | `2-ood/setup-ood.sh` |
| OIDC session passphrase | login node | `/etc/ood/oidc-crypto.passphrase` | root | `2-ood/setup-ood.sh` |
| Portal config with the OIDC secret | login node | `/etc/ood/config/ood_portal.yml` | 600 | `2-ood/setup-ood.sh` |
| Keycloak client `ood` secret | login node | `OOD_OIDC_SECRET` | root 0600 | `6-keycloak/2-realm.sh` (pushed from master) |
| MLflow proxy secret (copy) | login node | `MLFLOW_PROXY_SECRET` | root 0600 | `3-mlflow/2-deploy.sh` |
| MLflow admin password, proxy secret, `auth.db` | master | `MLFLOW_DATA/` | folder 700 | `3-mlflow/2-deploy.sh` |
| Keycloak passwords | master | `KEYCLOAK_DATA/*.pass`, `/etc/keycloak/*.env` | root 0600 | `6-keycloak/1-install.sh` |
| Admin page ssh key | login node | `ADMIN_KEY` | 600, owner `OOD_ADMIN` | `8-admin/1-install.sh` |
| Users' MLflow tokens | shared `/home` | `~/.mlflow/credentials` | the user's only | `3-mlflow/3-sync-tokens.sh` |
| Users' API keys | Kubernetes | a SHA-256 hash in the user's namespace | not readable by users | Model Hub |

**How secrets move:** never on a command line, where `ps` would show them. Scripts use `curl -K <(printf …)`,
`ldappasswd -S` (prompt) and `ldap… -y <file>`. Files are created under `umask 077`. Master pushes a secret to the
login node over ssh on standard input.
