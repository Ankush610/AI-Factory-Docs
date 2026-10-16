# 5 · Model Hub and gateway

> Users deploy their registered MLflow models (classic ML, deep learning, LLMs) as endpoints that run as Slurm jobs
> in their own Kubernetes namespace, manage them from a portal page, and call them through one HTTPS gateway with a
> personal API key.

**Folder:** [`OOD-Install/5-model-hub`](https://github.com/Ankush610/OOD-Install/tree/main/5-model-hub) ·
**Explained in:** [Model Hub](../../how-it-works/09-model-hub.md), [Gateway and API keys](../../how-it-works/10-gateway-and-api-keys.md) ·
**Known issues:** [Model Hub](../../known-issues/model-hub.md)

## What it sets up

- **Per-user tenancy:** namespace `u-<user>` with two admission policies (pods must run as the owner's user ID; pods
  must carry a Slurm time limit), a GPU quota, read-only model volumes, and a personal Kubernetes login
  (`~/.kube/aistack.config`, 1 year).
- **GPU plumbing:** one DRA DeviceClass per Slurm GPU type, and the `DynamicResources` plugin in slurm-bridge's
  scheduler.
- **Serving images:** `mlflow-serve-ml` and `mlflow-serve-torch` (library versions pinned to the training images),
  and a pinned mirror of `vllm-openai`.
- **The portal app** **Interactive Apps → Model Hub**, running as the logged-in user.
- **The gateway:** nginx + a key checker in namespace `model-hub`, on `GATEWAY_IP:443` (MetalLB), with a self-signed
  Rudra CA or the customer's certificate.
- **The public-models timer** (root, every minute) and the `model-register` command for LLM folders.

## Before you start

| Script | Runs on | Root |
|---|---|---|
| `1-setup.sh` | master | yes |
| `2-sync-users.sh` | master | yes |
| `3-test-tenancy.sh <userA> <userB>` (test) | master | yes |
| `4-images.sh` | master, with internet | **no** (rootless podman) |
| `5-test-serving.sh` (test) | master | **no** |
| `6-test-endpoints.sh <user> [ml] [llm] [torch] [--keep]` (test) | master | yes |
| `7-install-ood-app.sh` | **login node** | yes |
| `8-gateway.sh` | master | yes |

**Done first:** [Registry](registry.md), [LDAP](ldap.md), [Portal](portal.md), [MLflow](mlflow.md) (users need
`~/.mlflow/credentials`).

**Cluster pieces OOD-Install doesn't install (it only checks or uses them):**

- [ ] **slurm-bridge** (Helm release in `BRIDGE_NS`) with a Slurm partition `BRIDGE_PARTITION`.
- [ ] **NVIDIA DRA driver** (`gpu.nvidia.com` ResourceSlices).
- [ ] **cert-manager**, unless you set `GATEWAY_TLS_SECRET`.
- [ ] **MetalLB** with a pool that contains `GATEWAY_IP`.
- [ ] **NFS:** `NFS_SERVER` exports `MODELS_ROOT` and users' homes (pods mount them as NFS volumes).
- [ ] **Apptainer** and training images. Each image must hold `/opt/base-constraints.txt` with `mlflow-skinny==` pinned.
- [ ] **On the login node:** `kubectl` binary and access to master's 6443; `ldapsearch`; SSSD users; `squeue`; Python 3 with PyYAML.
- [ ] **Internet on master** for `4-images.sh` and `8-gateway.sh` (PyPI, PyTorch wheels, Docker Hub).

## Settings used

| Variable | Example | Meaning |
|---|---|---|
| `MODELS_ROOT`, `MODELHUB_USER`, `MODELHUB_UID` | `/home/apps/models`, `modelhub`, `951` | admin base models and public copies, and their owner |
| `BRIDGE_PARTITION`, `BRIDGE_NS` | `slurm-bridge`, `slurm` | Slurm partition for endpoints; slurm-bridge's namespace |
| `USER_GPU_QUOTA` | `1` | GPUs one user's endpoints may hold at once |
| `ENDPOINT_HOURS_DEFAULT`, `ENDPOINT_HOURS_MAX` | `24`, `96` | run time default and maximum (enforced by the cluster) |
| `NFS_SERVER` | `$MASTER_IP` | serves `/home` to pods |
| `ML_TRAIN_SIF`, `TORCH_TRAIN_SIF` | training images | the serving images are built to match these |
| `VLLM_VERSION` | `v0.30.0` | vLLM image tag. Never `latest` |
| `GATEWAY_NS`, `GATEWAY_IP`, `GATEWAY_HOST` | `model-hub`, `<gateway-ip>`, `""` | gateway namespace, address and optional DNS name |
| `GATEWAY_TLS_SECRET`, `GATEWAY_CA` | `""`, `/home/apps/gateway/ca.crt` | `""` = self-signed Rudra CA, written to `GATEWAY_CA` |
| `GATEWAY_NGINX_IMAGE`, `GATEWAY_PY_IMAGE` | pinned mirrors | the gateway's two images |
| `SSH_JUMP` | `<jump-host>` | shown in the API tab's tunnel hint |

Fixed in the code: endpoint port 8080, at most 8 GPUs per endpoint, gateway key cache 5 s, request body limit 100 MB,
proxy timeout 600 s.

## Install

Run in this order: **0 → 1 → 2 → (3) → 4 → (5, 6) → 7 → 8 → 2 → 7**. Every script is safe to rerun.

### 0. Let slurm-bridge manage namespaces by label (once, by hand)

Not scripted. On master, as root:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl label ns slurm-bridge aistack/slurm-bridge=true --overwrite
helm upgrade slurm-bridge oci://ghcr.io/slinkyproject/charts/slurm-bridge --version <deployed version> -n slurm \
  --reuse-values --set-json 'admission.managedNamespaceSelector={"matchLabels":{"aistack/slurm-bridge":"true"}}'
kubectl -n slurm rollout restart deploy/slurm-bridge-admission deploy/slurm-bridge-controllers deploy/slurm-bridge-scheduler
```

**Check:** `kubectl -n slurm get cm slurm-bridge-config -o yaml` shows `managedNamespaceSelector`.

!!! warning "After every `helm upgrade` of slurm-bridge"
    Rerun `1-setup.sh`. The upgrade resets the scheduler settings, and model pods then get a Slurm job but **no GPU
    inside**.

### 1. Cluster setup (master)

```bash
sudo bash 5-model-hub/1-setup.sh
```

| Step | What it does |
|---|---|
| Checks | master node exists, the `BRIDGE_PARTITION` exists, `gpu-classes.py` self-test |
| Account | creates `modelhub` (951) and `MODELS_ROOT`, `MODELS_ROOT/base` |
| Pod policies | applies `onboarding/uid-policy.yaml` (user-ID and time-limit policies) |
| GPU classes | one DeviceClass per Slurm GPU type, matched to the DRA product name |
| Scheduler | adds `DynamicResources` to slurm-bridge's `scheduler-config` if missing, restarts only the scheduler |
| `model-register` | installs it to `/home/apps/bin` with the site values filled in |
| Public-models timer | installs `public-models.py`, enables `model-hub-public.timer` (every minute) |
| Check | lists the policies and DeviceClasses; warns if `model-register` has an unfilled `@VAR@` |

**Expect:** the policies listed and one DeviceClass per GPU type. With a single GPU type, its `MAPS` column shows
`nvidia.com/gpu`. Then: "Next: put base models under …/base … then 2-sync-users.sh".

### 2. Users (master)

```bash
sudo bash 5-model-hub/2-sync-users.sh            # every LDAP user
sudo bash 5-model-hub/2-sync-users.sh <user>     # one user
```

Run it right after `1-setup.sh`: the time-limit policy needs each namespace's `aistack/max-hours` label. For each
user it creates `~/models` (700), the namespace with its labels and RoleBindings, the two read-only volumes
(`models-ro`, `my-models`), the GPU quota, and a client certificate in `~/.kube/aistack.config` (kept while it has
more than 30 days left).

**Check:**

```bash
sudo -u <user> kubectl --kubeconfig ~<user>/.kube/aistack.config get pods   # "No resources found"
```

### 3. Tenancy test (optional, master)

```bash
sudo bash 5-model-hub/3-test-tenancy.sh <userA> <userB>
```

Acting as userA, it proves: RBAC (own namespace only, can't relabel it, can't create volumes); the user-ID policy
(another ID, root, or no `securityContext` → refused); the time-limit policy (none, too long, or `exclusive: "true"`
→ refused); slurm-bridge picks up a pod; the GPU quota blocks a second GPU and direct ResourceClaims.
**Expect:** `Result: N passed, 0 failed`.

### 4. Serving images (master, no root)

```bash
bash 5-model-hub/4-images.sh
```

For `mlflow-serve-ml` and `mlflow-serve-torch`: reads the training image's package list, pins the serving image to
it, tags it `<mlflow version>-<hash of the folder>`, builds and pushes it if that tag is missing. Then mirrors
`vllm-openai:$VLLM_VERSION`, builds the newer per-image `serve-<sif>` images, and writes `images/built.env`.

**Expect:** the contents of `built.env`; existing tags say "already in …".

### 5–6. Serving tests (optional, master)

```bash
bash 5-model-hub/5-test-serving.sh                               # local podman: 5 classic-ML models → PASS
sudo bash 5-model-hub/6-test-endpoints.sh <user> ml llm torch    # real endpoints as <user>
```

`6-test-endpoints.sh` needs `LLM_PATH` for `llm` (default `MODELS_ROOT/base/qwen/Qwen3.5-0.8B`); `TORCH_GPUS=0`
runs the torch test on CPU. **Expect:** `Result: N passed, 0 failed`; endpoints are deleted unless `--keep`.

### 7. The portal app (login node)

```bash
sudo bash 5-model-hub/7-install-ood-app.sh
```

Needs `images/built.env` and the `images/*/constraints.txt` files that `4-images.sh` writes into **the same
checkout**. It copies the app and `render.py`, copies the gateway CA (self-signed sites), and writes `site.json`.
**Expect:** `passenger_wsgi.py: OK`. Restart Web Server; open **Interactive Apps → Model Hub**.

### 8. The gateway (master)

```bash
sudo bash 5-model-hub/8-gateway.sh
```

| Step | What it does |
|---|---|
| Checks | master node, cert-manager (or `GATEWAY_TLS_SECRET`), a MetalLB pool, `GATEWAY_IP` not taken, `auth.py` self-test |
| Images | mirrors the nginx and Python images if missing |
| Certificate | creates the namespace; applies the Rudra CA and gateway certificate (or checks the customer's Secret) |
| Gateway | puts `nginx.conf` and `auth.py` in a ConfigMap, applies `gateway.yaml`, waits for the rollout |
| Check | writes `GATEWAY_CA`; expects `/healthz` → **200** and a call with no key → **401** |

**Then:** rerun `2-sync-users.sh` on master (gives the gateway read rights in every user namespace) and
`7-install-ood-app.sh` on the login node, then Restart Web Server.

New people are handled automatically: `1-ldap/add-user.sh` and `group.sh` run `2-sync-users.sh` for them.

## Everyday tasks

On master, as root, unless noted.

### Limits

```bash
sudo bash 5-model-hub/2-sync-users.sh --show                 # everyone's limits
sudo bash 5-model-hub/2-sync-users.sh bob --gpus 2           # one person: 2 GPUs
sudo bash 5-model-hub/2-sync-users.sh bob --max-hours 240    # one person: longer runs
sudo bash 5-model-hub/2-sync-users.sh bob --reset            # back to site.conf
```

- Changed `USER_GPU_QUOTA` or `ENDPOINT_HOURS_MAX`: rerun `2-sync-users.sh` (all users).
- Changed `ENDPOINT_HOURS_DEFAULT`: rerun `7-install-ood-app.sh` on the login node.
- Exceptions survive reruns. Running endpoints keep the time they started with.
- The Admin page → Users → **Limits** does the same.

### Other tasks

| Task | How |
|---|---|
| Add base models | copy the Hugging Face folder to `MODELS_ROOT/base/<org>/<model>` (as `modelhub`); as yourself (not sudo) `model-register $MODELS_ROOT/base/<org>/<model> <name>`; then **Make public** in Model Hub |
| Upgrade serving images | `bash 4-images.sh` (as yourself); optionally the tests; then `7-install-ood-app.sh` on the login node. Running endpoints keep their old image. If `ML_TRAIN_SIF` changed, also rerun `1-setup.sh` |
| Update the portal app | `7-install-ood-app.sh` on the login node |
| Update the gateway (config, code, images, IP) | `8-gateway.sh`. The pod rolls when the config changes |
| Renew users' Kubernetes certificates | rerun `2-sync-users.sh` regularly; it renews certificates with < 30 days left. **Nothing schedules this** |
| Gateway certificate renewed by cert-manager | `kubectl -n model-hub rollout restart deploy/gateway` (nginx reads it only at start) |
| Stop someone's endpoint | Admin page → Running → Stop (keeps the Service, the owner sees why); or `kubectl -n u-<user> delete pod <ep>` |
| Teams | `1-ldap/group.sh …`; members are re-synced; the gateway sees it within seconds |

### Look around

```bash
kubectl get ns -l aistack/user                            # user namespaces
kubectl get svc,pods -A -l model-hub/runtime              # every endpoint
squeue -p slurm-bridge                                    # their Slurm jobs
kubectl -n model-hub logs deploy/gateway -c nginx         # JSON access log
kubectl -n model-hub logs deploy/gateway -c auth          # one JSON line per key check
systemctl status model-hub-public.timer; journalctl -u model-hub-public -n 50
```

## Troubleshooting

### Install

| Symptom | Cause | Fix |
|---|---|---|
| `No Slurm partition slurm-bridge.` | `BRIDGE_PARTITION` doesn't exist | create it, or fix `BRIDGE_PARTITION` |
| `No GPU class made` | no GPU node up (no ResourceSlices), or Slurm's type name isn't in the DRA product name | start a GPU node and rerun; for odd names, create the DeviceClass by hand |
| `no multiPoint: in scheduler-config; … add DynamicResources by hand` | a new slurm-bridge chart changed the layout | edit `scheduler-config` in `BRIDGE_NS` by hand, restart `slurm-bridge-scheduler` |
| `model-register still has an unfilled @VAR@` | a missing `site.conf` value | fix `site.conf`, rerun |
| `1-setup.sh` step 7 fails with the journal | the first public-models run failed (MLflow down, `admin.pass` missing) | fix MLflow, rerun |
| `UID 951 already belongs to …` | `MODELHUB_UID` taken | pick a free ID below `MIN_UID` |
| `2-sync-users.sh`: `Run 1-setup.sh first` | no user-ID policy yet | run `1-setup.sh` |
| `certificate for <user> not issued` | the Kubernetes CSR wasn't signed within 20 s | `kubectl get csr`; check the signer; rerun |
| `SKIP <user> (not resolvable)` | SSSD on master doesn't know them | check SSSD |
| `<sif> has no mlflow-skinny pin` | the training image lacks the pin | rebuild the image with it |
| `build of <img> failed 3 times` | network drops, or a version conflict (`verify-pins.py` BAD lines) | rerun (the pip cache keeps downloads); read the BAD lines |
| `<sif> already has …/sitecustomize.py` | the image ships its own | merge the PyTorch device fix into it by hand |
| `7-install-ood-app.sh`: `No images/built.env` / can't read `constraints.txt` | `4-images.sh` didn't run in this checkout | run it here, or copy the files |
| `No cert-manager …` / `No MetalLB address pool` | missing cluster piece | install it, or set `GATEWAY_TLS_SECRET` |
| `GATEWAY_IP … is already used` | another LoadBalancer has it | pick a free IP from the pool |

### Tenancy, policies and GPUs

| Symptom | Cause | Fix |
|---|---|---|
| GPU pod runs with a Slurm job, but no `/dev/nvidia*` / vLLM `Failed to infer device type` | a slurm-bridge `helm upgrade` dropped `DynamicResources` | `1-setup.sh`, then redeploy |
| Test pod stays `Pending`, no Slurm job | namespace not labelled, or slurm-bridge not on label selection | check `managedNamespaceSelector` (step 0) |
| `pods need annotation slurmjob.slinky.slurm.net/timelimit` | a pod with no time limit | Model Hub sets it; with `kubectl`, add it (minutes) |
| `time limit N min is over your maximum of H hours` | asked for more than `max-hours` | fewer hours, or `--max-hours` for that user |
| `this namespace has no aistack/max-hours label` | `2-sync-users.sh` hasn't run since `1-setup.sh` | run it |
| `exceeded quota: gpus` | the user's GPUs are in use | stop one, or `--gpus 2` |
| `denied request: … pod-runs-as-namespace-owner` | the pod's user ID isn't the owner's | render with the owner's `uid`/`gid` |
| Endpoint runs past its hours; `TimeLimit=365-00:00:00` | the pod was wrapped in a Deployment (slurm-bridge reads the top owner) | endpoints must be bare Pods |
| User's `kubectl`: `Unauthorized` | certificate expired or cluster CA changed | `2-sync-users.sh <user>` |
| Deploy: `kubectl …: command not found` / `connection refused` | no `kubectl` on the login node, or 6443 blocked | install it; open 6443 for `LOGIN_IP` |

### Gateway

| Symptom | Cause | Fix |
|---|---|---|
| 403 for every call right after install | `2-sync-users.sh` hasn't run since `8-gateway.sh` | run it |
| `{"error":"unknown or revoked API key"}` (401) | wrong or old key, or a brand-new key | copy it again; wait a few seconds after making one |
| `{"error":"no such endpoint, or it isn't shared with you"}` (403) | wrong URL, deleted endpoint, or not shared | use the URL from the API tab; the owner checks **Share** |
| Shared with a team, member still gets 403 | their namespace doesn't list the team yet | `2-sync-users.sh <member>` |
| 502 / 504 | endpoint stopped, loading, or waiting for a GPU | check its state in Model Hub |
| `503 gateway can't check keys` | the key checker can't reach the Kubernetes API | `kubectl -n model-hub logs deploy/gateway -c auth` |
| `curl: (60) SSL certificate problem` | the Rudra CA isn't trusted | `curl --cacert gateway-ca.crt` |
| "Shared with me": `gateway didn't answer` | no port 8444 yet, or it's blocked | rerun `8-gateway.sh`; `curl -k https://<gateway-ip>:8444/shared` should give 400 |

### The app and public models

| Symptom | Cause | Fix |
|---|---|---|
| `No ~/.mlflow/credentials …` (409) | no MLflow token | `3-mlflow/3-sync-tokens.sh` |
| `No ~/.kube/aistack.config …` (409) | user not synced | `2-sync-users.sh <user>` |
| Share: `no such team` | not in LDAP, or the login node can't query LDAP | `group.sh create <team>`; test `ldapsearch` from the login node |
| No CA download on the API key page | `GATEWAY_CA` not copied | rerun `8-gateway.sh`, then `7-install-ood-app.sh` |
| Deploy: `<name> is still running` | pods can't be changed in place | delete it first, or use another name |
| Deploy: `you may use N GPU(s)` / `run time must be 1..H hours` | over the limits | choose within limits, or set an exception |
| Deploy: `<path> is not under MODELS_ROOT or ~/models` | LLM registered from elsewhere | move it under `~/models`, register again |
| Endpoint "failed": `ImagePullBackOff` | the node can't pull from the registry | check the registry setup on that node |
| A user can't see a public model | they have no MLflow READ grant yet (given within a minute, only to users MLflow knows) | `journalctl -u model-hub-public -n 50` |
| `… is public, but its files are still being copied` / `couldn't be shared` | the timer isn't done, or the copy failed | wait; check the journal (disk space, folder owner, experiment owner) |
| Page is plain unstyled HTML | `index.html` was moved into `public/` | keep it in the app root |
| `model-register`: `not under ~/models/` / `is a LoRA adapter` / `has no config.json + *.safetensors` / `Run as yourself` | input checks | follow the message (merge LoRA, `save_pretrained`, no sudo) |

### Model logging gotchas

| Symptom | Cause | Fix |
|---|---|---|
| Torch endpoint: `… on 'cpu' device, it can't be loaded on 'cuda'` | MLflow 3.x pt2 models load only on the export device | fixed in the image; rebuild with `4-images.sh` |
| `mlflow.transformers.save_model`: `No module named 'torchvision'` | MLflow always adds torchvision for torch models | pass `pip_requirements=[…]`, or add torchvision to the image |
| `mlflow.keras.save_model(…, input_example=…)`: `FileNotFoundError` | MLflow 3.15.1 writes the example before the folder | `os.makedirs(path)` first, or use `log_model` |
| `Cannot set a deleted experiment 'mh-test-<user>'` | the experiment is in MLflow's trash | restore it in the MLflow UI |
