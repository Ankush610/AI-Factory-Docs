# 3 · MLflow

> One always-on MLflow 3.15.1 server on Kubernetes for everyone. People log in with their cluster password (or
> through the portal with Keycloak), jobs log in with a per-user token, and each user sees only their own work.

**Folder:** [`OOD-Install/3-mlflow`](https://github.com/Ankush610/OOD-Install/tree/main/3-mlflow) ·
**Explained in:** [MLflow](../../how-it-works/08-mlflow.md) ·
**Known issues:** [MLflow](../../known-issues/mlflow.md)

## What it sets up

- **The server:** Deployment `mlflow` (1 replica) in namespace `mlflow`, pinned to master, NodePort `MLFLOW_PORT`
  (30500). It runs as the local `mlflow` account (950) and keeps its data in `MLFLOW_DATA` on master's disk.
- **The login plugin** (`ldap_auth.py`): the portal's proxy header (Keycloak mode), a password or token stored in
  MLflow's own `auth.db`, or an LDAP bind. Default permission is **none**: everything is private to its creator.
- **Job tokens:** a random token per LDAP user in `~/.mlflow/credentials`. It survives password changes.
- **The portal page** **Interactive Apps → MLflow (shared)**: server status, the tracking URI, the naming rule, an
  FAQ, and **Open MLflow**.

## Before you start

| Script | Runs on | Root |
|---|---|---|
| `1-build-image.sh` | master | **no** (needs podman and push access to the registry) |
| `2-deploy.sh` | master | yes |
| `3-sync-tokens.sh` | master | yes (it reads LDAP on `localhost`) |
| `4-install-ood-app.sh` | login node | yes |

- **Done first:** [Registry](registry.md), [LDAP](ldap.md) (`LDAP_CA` must exist), [Portal](portal.md).
- **Kubernetes:** `kubectl` as root with `/etc/kubernetes/admin.conf`; master registered as `MASTER_HOST`.
- **Root SSH** from master to the login node, to copy the proxy secret.
- **Internet on master at build time:** `python:3.12-slim` and pinned packages from PyPI.
- **Port 30500** on master open for `LOGIN_IP` and the compute nodes. In `ldap` mode, users' browsers must reach it
  too. This folder opens no ports.

## Settings used

| Variable | Example | Meaning |
|---|---|---|
| `MLFLOW_VERSION` | `3.15.1` | must equal `mlflow==` in `image/requirements.txt` |
| `MLFLOW_IMAGE` | `$REGISTRY/mlflow-server:$MLFLOW_VERSION-sso` | server image |
| `MLFLOW_PORT` | `30500` | NodePort on `MASTER_IP` |
| `MLFLOW_PREFIX` | `/node/$MASTER_IP/$MLFLOW_PORT` | URL path MLflow serves under. Must be under `/node/` for single sign-on |
| `MLFLOW_URI` | `http://$MASTER_IP:$MLFLOW_PORT$MLFLOW_PREFIX` | the tracking URI |
| `MLFLOW_DATA` | `/home/apps/mlflow-shared` | database, users, artifacts |
| `MLFLOW_USER`, `MLFLOW_UID` | `mlflow`, `950` | the account that owns the data and runs the pod |
| `MLFLOW_PROXY_SECRET` | `/etc/ood/mlflow-proxy.secret` | the secret's copy on the login node |
| `OOD_AUTH` | `keycloak` | `keycloak` = Open MLflow goes through the portal, already logged in; `ldap` = straight to the port |

Fixed in the code: container port 5000, `--workers 1`, CPU/memory 1/1Gi requests and 2/2Gi limits, login cache
300 s, token length 48 hex characters.

## Install

### 1. Build and push the image (master, no root)

```bash
bash 3-mlflow/1-build-image.sh
# optional: regenerate the pins from an existing venv first
bash 3-mlflow/1-build-image.sh --from-venv /path/to/venv
```

It stops if `mlflow==` in `requirements.txt` differs from `MLFLOW_VERSION`, then builds, pushes and smoke-tests
the image.

**Expect:** `ldap3 + ldap_auth.py: OK`, and a tag list containing `3.15.1-sso`.

### 2. Deploy (master)

```bash
sudo bash 3-mlflow/2-deploy.sh
# only if admin.pass was lost but auth.db exists:
sudo ADMIN_PASS='<MLflow admin password>' bash 3-mlflow/2-deploy.sh
```

| Step | What it does |
|---|---|
| `== 1. Checks` | `LDAP_CA` exists, `kubectl get node $MASTER_HOST` works, the image is in the registry |
| `== 2.` | creates the `mlflow` account (950, no login) and `MLFLOW_DATA/artifacts`, owner 950, mode 700 |
| `== 3.` | writes `admin.pass` once, `basic_auth.ini` every time, `proxy.secret` once; copies `proxy.secret` to the login node over root SSH |
| `== 4.` | fills `mlflow.yaml` with `envsubst`, applies it, creates the Flask session Secret once, restarts the deployment (waits up to 180 s) |
| `== 5. Check` | three HTTP codes |

**Expect:**

```text
health        -> 200   (expect 200)
api, no login -> 401   (expect 401)
api, admin    -> 200   (expect 200)
```

!!! note "Keycloak mode"
    If the portal's Apache config has no MLflow secret header yet, the script says
    `OOD_AUTH=keycloak: also rerun ../2-ood/setup-ood.sh on <LOGIN_HOST>`. Do that: the portal was installed
    before this secret existed.

### 3. Job tokens for every LDAP user (master)

```bash
sudo bash 3-mlflow/3-sync-tokens.sh
```

For each LDAP user it keeps a token that still works, otherwise it makes a new one and writes
`~/.mlflow/credentials` (folder 700, file 600, owned by the user).

**Expect:** one line per user: `ok`, `added` or `token`. `SKIP` or `FAILED` lines need attention.

`1-ldap/add-user.sh` runs this automatically for new people when MLflow answers.

### 4. The portal page (login node)

```bash
sudo bash 3-mlflow/4-install-ood-app.sh
```

**Expect:** `passenger_wsgi.py: OK`. Click **Restart Web Server**, open **Interactive Apps → MLflow (shared)**.
Both status rows show **OK**.

### Check it works

```bash
# on master, as root
kubectl --kubeconfig /etc/kubernetes/admin.conf -n mlflow get pod,svc
( source site.conf; curl -s -o /dev/null -w '%{http_code}\n' "$MLFLOW_URI/health" )   # 200
```

## Everyday tasks

On master as root. Use `--kubeconfig /etc/kubernetes/admin.conf` (or set `KUBECONFIG`).

| Task | How |
|---|---|
| Add a person | `1-ldap/add-user.sh <name>` (also makes the token). If MLflow was down: `3-sync-tokens.sh` later |
| Remove a person | `1-ldap/remove-user.sh <name>`: always deletes their MLflow login. Their work stays, private, tagged `removed_owner=<name>`, unless `--delete-mlflow` |
| Fix job tokens | `sudo bash 3-mlflow/3-sync-tokens.sh` (replaces only tokens that stopped working) |
| Force a new token for one user | `sudo rm ~<name>/.mlflow/credentials && sudo bash 3-mlflow/3-sync-tokens.sh` |
| Log in as the MLflow admin | user `admin`, password in `sudo cat /home/apps/mlflow-shared/admin.pass` |
| Logs / restart / status | `kubectl -n mlflow logs deploy/mlflow` · `rollout restart deploy/mlflow` · `get pod,svc` |
| Empty everyone's trash | `kubectl -n mlflow exec deploy/mlflow -- mlflow gc --backend-store-uri sqlite:////data/mlflow.db --artifacts-destination /data/artifacts` |
| After a portal page change | `4-install-ood-app.sh` on the login node, then Restart Web Server |

### Back up

Back up `MLFLOW_DATA` **as one piece**: `mlflow.db`, `auth.db` and `artifacts/` together, plus `admin.pass` and
`proxy.secret`. Losing `mlflow.db` loses every run, model version and alias.

!!! tip
    For a consistent copy, stop the server first
    (`kubectl -n mlflow scale deploy/mlflow --replicas=0`), copy, then scale back to 1. The repo doesn't
    prescribe this; see [known issues](../../known-issues/mlflow.md#backup-steps-are-incomplete).

### Upgrade

```bash
vi site.conf                              # MLFLOW_VERSION=<new>
vi 3-mlflow/image/requirements.txt        # mlflow==<new>  (or: 1-build-image.sh --from-venv <venv>)
bash 3-mlflow/1-build-image.sh            # master, no root
sudo bash 3-mlflow/2-deploy.sh            # master
```

The server must stay at or above the MLflow client (`mlflow-skinny`) in the training images. The database upgrades
itself, one way only.

### Rotate the proxy secret

```bash
sudo rm /home/apps/mlflow-shared/proxy.secret
sudo bash 3-mlflow/2-deploy.sh            # master: new secret, copied to the login node, pod restarted
sudo bash 2-ood/setup-ood.sh              # login node: Apache picks it up
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `requirements.txt pins mlflow==X, site.conf says …=Y` | the two versions differ | make them equal |
| `THESE PACKAGES DO NOT MATCH THE HASHES` | PyPI dropped the connection | rerun |
| `2-deploy.sh`: `is not in the registry` | image not built, or another tag in `site.conf` | `bash 1-build-image.sh` |
| `No /home/apps/ldap/ca.crt` | LDAP not installed | `1-ldap/1-server.sh` |
| `UID <n> already belongs to '<x>'` | `MLFLOW_UID` taken on master | pick a free ID below `MIN_UID` |
| `auth.db already exists: rerun with ADMIN_PASS=…` | `admin.pass` lost | `sudo ADMIN_PASS='<pw>' bash 2-deploy.sh` |
| Rollout timed out: `failed calling webhook "pods.slinky.slurm.net" … connection refused` | slurm-bridge's admission webhook is down; it blocks new pods everywhere except `kube-system`/`slurm` | give its pods a toleration for `slinky.slurm.net/managed-node` (put it in the Helm values, or `helm upgrade` loses it) |
| `namespace mlflow … is being terminated` | a broken aggregated API (e.g. KEDA's metrics) blocks the delete | once empty: `kubectl get ns mlflow -o json \| jq ".spec.finalizers=[]" \| kubectl replace --raw /api/v1/namespaces/mlflow/finalize -f -` |
| Check: `health` or `api, admin` not 200 | pod not ready, wrong prefix, or admin password mismatch | `kubectl -n mlflow logs deploy/mlflow`; `describe pod -l app=mlflow` |
| `could not copy the proxy secret` / portal: `MLflow SSO skipped: no …mlflow-proxy.secret` | master's root SSH copy to the login node failed | fix root SSH, rerun `2-deploy.sh`, then `setup-ood.sh` |
| Keycloak on, but **Open MLflow** still asks for a password | page installed before the switch, or Apache has no secret header | `2-deploy.sh` (master), then `setup-ood.sh` and `4-install-ood-app.sh` (login node), Restart Web Server |
| Through the portal, MLflow answers 401 | `proxy.secret` changed after `setup-ood.sh`, or the pod runs an old image | rerun `2-deploy.sh`, then `setup-ood.sh`; check the pod's image ends in `-sso` |
| MLflow page refuses the SSH password | opened through `/node/…` without single sign-on, user not in LDAP, LDAP unreachable from the pod, or password changed in the last 5 min | use **Open MLflow**; test with `ldapwhoami`; pod logs; wait 5 min |
| Open MLflow: page can't be reached (test cluster) | the laptop tunnel lacks the MLflow port | add `-L 30500:<master-ip>:30500` |
| `3-sync-tokens.sh`: `does not answer or admin login fails` | pod down, or `admin.pass` doesn't match | check the pod; rerun `2-deploy.sh` |
| `No users in LDAP` | no people yet, or not run on master | add users; run on master |
| `SKIP <user> (not resolvable, is SSSD running?)` | `getent passwd` can't resolve them | `systemctl status sssd` on master |
| `FAILED <user> (MLflow)` | MLflow refused both create and update | pod logs; try as admin in the UI |
| Job gets `401` | token missing or stale | `3-sync-tokens.sh` |
| Job: `KeyError: 'sqlite'` | the training images' `mlflow-skinny` can't open SQLite | always use the HTTP tracking URI |
| Portal page: **MLflow server: Problem** | `/health` didn't answer within 3 s from the login node | check the pod and that 30500 is open for `LOGIN_IP` |
| Portal page: **Job login: Problem** | `~/.mlflow/credentials` missing | `3-sync-tokens.sh` |
| A removed person still opens MLflow for a few minutes | 5-minute login cache | wait; they have no rights left |
| Alert `ServiceUnreachable` for mlflow | the `/health` probe failed | `kubectl get pods -n mlflow` |
