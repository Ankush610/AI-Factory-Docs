# Model Hub and gateway

Issues in [`5-model-hub`](https://github.com/Ankush610/OOD-Install/tree/main/5-model-hub). How to read this page:
[Known issues](index.md#how-to-read-these-pages).

| Issue | Type | Confidence |
|---|---|---|
| [No network lock: pods can skip the gateway](#no-network-lock-pods-can-skip-the-gateway) | Security | Confirmed |
| [The gateway can read every user's MLflow token](#the-gateway-can-read-every-users-mlflow-token) | Security | Inferred |
| [The GPU type a user picks is ignored](#the-gpu-type-a-user-picks-is-ignored) | Bug | Inferred |
| [Gateway cache time is documented three ways](#gateway-cache-time-is-documented-three-ways) | Docs | Confirmed |
| [Deployed images don't set `STOPSIGNAL SIGINT`](#deployed-images-dont-set-stopsignal-sigint) | Bug | Confirmed |
| [Certificates aren't renewed automatically](#certificates-arent-renewed-automatically) | Gap | Confirmed |
| [A separate login node can't see the image files](#a-separate-login-node-cant-see-the-image-files) | Gap | Inferred |
| [No firewall rule for the gateway](#no-firewall-rule-for-the-gateway) | Gap | Confirmed |
| [No backup or restore for the Kubernetes side](#no-backup-or-restore-for-the-kubernetes-side) | Gap | Confirmed |
| [Built but unused: `serve-*` images and LoRA](#built-but-unused-serve-images-and-lora) | Gap | Confirmed |
| [Stale or misplaced README text](#stale-or-misplaced-readme-text) | Docs | Confirmed |
| [Open questions](#open-questions) | Question | — |

## No network lock: pods can skip the gateway

- **Type:** Security · **Confidence:** Confirmed · **Where:** `5-model-hub/README.md:64-65`, `gateway/nginx.conf:52`
- **What happens:** any pod in the cluster can reach an endpoint's ClusterIP directly, without a key or sharing
  check. Classic-ML endpoints have no key of their own. The fix is a network policy (Calico or Cilium instead of
  flannel), roadmap step 1.

## The gateway can read every user's MLflow token

- **Type:** Security · **Confidence:** Inferred · **Where:** `gateway/gateway.yaml:23-25`, `render.py:75-76`
- **What happens:** the gateway's ServiceAccount may `get secrets` in every user namespace. That includes
  `mlflow-creds`, which holds each user's MLflow token. A compromised gateway pod could read them.
- **Fix:** limit the Role with `resourceNames` to `model-hub-key` and the `*-key` Secrets.

## The GPU type a user picks is ignored

- **Type:** Bug · **Confidence:** Inferred · **Where:** `render.py:48-61`, `gpu-classes.py:32-33,90-91`
- **What happens:** `render.py` requires a GPU type but never writes it into the Pod; it only requests
  `nvidia.com/gpu: N`. On a cluster with one GPU type this works. With several types, `nvidia.com/gpu` maps to no
  DeviceClass at all, so GPU endpoints can't be scheduled.

## Gateway cache time is documented three ways

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `README.md:54`, `app.js:394,845`, `auth.py:45`, `gateway.yaml:92`
- **What happens:** the README says 60 s, the Share dialog says "within a minute", another hint says 5 s. The code
  uses **5 s**. These docs now say "a few seconds".

## Deployed images don't set `STOPSIGNAL SIGINT`

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `4-images.sh:52`; `images/mlflow-serve-*/Containerfile`
- **What happens:** `mlflow` as process 1 ignores SIGTERM. The README says images built by `4-images.sh` stop with
  SIGINT, but only the unused `serve-*` images do. `mlflow-serve-ml` and `mlflow-serve-torch`, which endpoints
  actually run, probably wait out the full grace period when stopped.

## Certificates aren't renewed automatically

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `2-sync-users.sh:55-95`, `gateway/tls.yaml:30`
- **What happens:**
    - Users' Kubernetes certificates last 1 year and are renewed only when someone reruns `2-sync-users.sh`.
      Nothing schedules it; when they expire, Model Hub stops working for that user (`Unauthorized`).
    - cert-manager renews the gateway's certificate, but nginx reads it only at start.
- **Workaround:** run `2-sync-users.sh` monthly (for example from cron), and restart the gateway after a renewal.

## A separate login node can't see the image files

- **Type:** Gap · **Confidence:** Inferred · **Where:** `7-install-ood-app.sh:15,39`, `.gitignore:2`
- **What happens:** `7-install-ood-app.sh` runs on the login node but reads `images/built.env` and the gitignored
  `images/*/constraints.txt` that `4-images.sh` writes on master. This works only when both run from the same
  checkout on the shared `/home`.

## No firewall rule for the gateway

- **Type:** Gap · **Confidence:** Confirmed
- **What happens:** nothing opens or limits `GATEWAY_IP:443/8444`, and the top-level README's port list doesn't
  mention them. Decide which networks may call models.

## No backup or restore for the Kubernetes side

- **Type:** Gap · **Confidence:** Confirmed
- **What happens:** API key hashes, share settings and limit exceptions live only in Kubernetes. After a cluster
  rebuild, users must make new API keys and owners must re-share endpoints; admins must set exceptions again.

## Built but unused: `serve-*` images and LoRA

- **Type:** Gap · **Confidence:** Confirmed
- `images/catalog.json` lists 8 `serve-<sif>` images (the planned v3), but nothing reads it.
- `render.py` supports LoRA (`base_path`), but `model-register` refuses adapters and the app never passes it.

## Stale or misplaced README text

- **Type:** Docs · **Confidence:** Confirmed
- `8-gateway.sh:4` and `site.conf:117` say only the owner can call an endpoint ("sharing: later"); sharing works.
- `README.md:77-78` puts the per-user volume and kubeconfig rows under "Limits" instead of "Per user".
- `README.md:35` refers to "revision 4 above", which doesn't exist.
- Only `7-install-ood-app.sh` checks its host; the README claims every script does.
- The README points to the v1 design docs; `CLAUDE.md` says the current scope is version 2.

## Open questions

- Which NFS exports and options does a production site need? `site.conf` calls `NFS_SERVER` "Dummy".
- Who should own base models? The timer falls back to `admin`, but `model-register` refuses root and says to register
  "as yourself".
- Model names containing `'` are skipped by the public-models timer and never get a status.
