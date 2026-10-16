# Install and docs

Issues that cross components: install order, prerequisites, shared conventions and the top-level READMEs. How to
read this page: [Known issues](index.md#how-to-read-these-pages).

| Issue | Type | Confidence |
|---|---|---|
| [First install order with Keycloak](#first-install-order-with-keycloak) | Bug | Confirmed |
| [Top-level README covers only folders 0–4](#top-level-readme-covers-only-folders-04) | Docs | Confirmed |
| [Not every script checks its host](#not-every-script-checks-its-host) | Docs | Confirmed |
| [`CLAUDE.md` is out of date](#claudemd-is-out-of-date) | Docs | Confirmed |
| [Prerequisites missing from the list](#prerequisites-missing-from-the-list) | Docs | Confirmed |
| [Other nodes also need package access](#other-nodes-also-need-package-access) | Docs | Confirmed |
| [Where the repo must live isn't stated](#where-the-repo-must-live-isnt-stated) | Docs | Confirmed |
| ["Add a compute node" is incomplete](#add-a-compute-node-is-incomplete) | Docs | Confirmed |
| [Certificate renewal isn't really documented](#certificate-renewal-isnt-really-documented) | Docs | Confirmed |
| [No backup or restore procedure](#no-backup-or-restore-procedure) | Gap | Confirmed |
| [Smaller items](#smaller-items) | Mixed | — |

## First install order with Keycloak

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `site.conf:42`, `2-ood/setup-ood.sh:28-32`, `6-keycloak/1-install.sh:30-32`
- **What happens:** `site.conf` ships `OOD_AUTH=keycloak`, but the portal can't be installed in that mode before
  Keycloak, and Keycloak needs the portal first. The order that works is in the
  [Admin guide](../admin/index.md#install-order). Details: [Portal](portal.md#first-install-cant-use-the-shipped-ood_authkeycloak).
- **Fix:** ship `OOD_AUTH=ldap` in `site.conf`, or say so in the top-level README.

## Top-level README covers only folders 0–4

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `README.md:3-8,22-29,43-73`
- **What happens:** the intro lists only LDAP, OOD, MLflow and VS Code; the folder table and install block stop at
  `4-vscode`. Folders 5–9 get one paragraph. Use the [Admin guide](../admin/index.md) for the full order.

## Not every script checks its host

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `README.md:73`
- **What happens:** the README says "each script says so if run on the wrong one". These have **no** host check:
    - `1-ldap/add-user.sh`, `set-password.sh`, `group.sh`, `import-local-users.sh`
    - `3-mlflow/1-build-image.sh`, `3-sync-tokens.sh`
    - `4-vscode/1-install-code-server.sh`
    - `5-model-hub/1-setup.sh` to `6-test-endpoints.sh`, `8-gateway.sh`
    - `7-jupyter/1-install-jupyter.sh`
- Most of them only work on master (they use `ldap://localhost` or `admin.conf`) and fail with a less clear message
  elsewhere. Use the "Runs on" column in [Scripts](../reference/scripts.md).

## `CLAUDE.md` is out of date

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `CLAUDE.md:32,47,63`
- **What happens:** it describes VS Code as a "Batch Connect" app with `form.yml`, `submit.yml.erb` and
  `view.html.erb`. None exist; it's a Passenger page. Its sanity check loads the missing `form.yml`. It also refers
  to a git tag `v1` that isn't in the repo.

## Prerequisites missing from the list

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `README.md:31-41`
- **What happens:** the scripts need these, but the top-level list doesn't name them:
    - Apptainer at `APPTAINER` and the training images in `CONTAINERS_ROOT` (nothing in OOD-Install installs them)
    - slurm-bridge with Helm, plus the one manual Helm change in `5-model-hub`
    - MetalLB with a pool containing `GATEWAY_IP`; cert-manager (or a customer certificate)
    - the NVIDIA driver and DRA driver on GPU nodes
    - `python3` and `envsubst` (gettext)
- The [Admin guide](../admin/index.md#before-you-start) lists them all.

## Other nodes also need package access

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `README.md:33`, `1-ldap/2-client.sh:15`, `2-ood/setup-ood.sh:37-42`, `9-monitoring/node-setup.sh:19-20`
- **What happens:** the README asks for internet on master only. But `2-client.sh` installs packages on **every**
  node, `setup-ood.sh` downloads repositories and RPMs on the login node, and `node-setup.sh` installs `curl` (and
  `python3`) on every node. Either those nodes need internet, or a local package mirror.

## Where the repo must live isn't stated

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `1-ldap/2-client.sh:4`, `8-admin/1-install.sh:25-26`
- **What happens:** compute and login nodes run scripts from the same checkout, so it must sit on the shared `/home`.
  The admin helper records its path, and the folder must become root-owned before anyone else gets admin rights. No
  doc names the path or owner.
- **Related:** `/home/apps/bin` is hardcoded in `0-registry` and `5-model-hub/1-setup.sh`, against the repo's own
  "never hardcode a path" rule.

## "Add a compute node" is incomplete

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `README.md:85`
- **What happens:** the README says only "run `1-ldap/2-client.sh` on it". The code needs more. The full checklist:
    1. Add it to `COMPUTE_NODES` in `site.conf`.
    2. `1-ldap/2-client.sh` on the new node.
    3. `0-registry/setup-registry.sh` on master (containerd trust).
    4. `4-vscode/2-slurm-viewer.sh` on master, if `slurm.conf` changed (it copies `slurm.conf` to every node).
    5. `9-monitoring/1-install.sh` on master (node_exporter and collectors).
    6. If it has GPUs: `5-model-hub/1-setup.sh` (GPU classes), then `4-vscode/3-install-ood-app.sh` and
       `7-jupyter/2-install-ood-app.sh` on the login node (GPU choices), then Restart Web Server.

## Certificate renewal isn't really documented

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `9-monitoring/README.md:122`
- **What happens:** the alert runbook says to renew certificates by rerunning the install scripts. That doesn't work
  for two of the three:

| Certificate | Rerun renews it? | What works |
|---|---|---|
| Portal (`2-ood`) | no: an existing certificate is never replaced | delete `OOD_CERT` and its key, rerun `setup-ood.sh`, then rerun `6-keycloak/1-install.sh` (inferred) — [details](portal.md#certificate-renewal-isnt-documented) |
| LDAP (`1-ldap`) | no: an existing instance is skipped | renew with 389 DS's own tools, copy the CA to `LDAP_CA` — [details](ldap.md#rerunning-1-serversh-doesnt-renew-the-certificate) |
| Gateway (`5-model-hub`) | cert-manager renews it | restart the gateway pod afterwards — [details](model-hub.md#certificates-arent-renewed-automatically) |

## No backup or restore procedure

- **Type:** Gap · **Confidence:** Confirmed
- **What happens:** `site.conf` mentions backing up the registry and the last-UID file, but no script or procedure
  exists for any of these:

| Data | Where | Note |
|---|---|---|
| LDAP | 389 DS + `/root/.ldap-last-uid` + `/root/.ldap-dm.pass` | [LDAP](ldap.md#no-backup-or-restore-procedure) |
| MLflow | `MLFLOW_DATA` (one piece) + `admin.pass`, `proxy.secret` | [MLflow](mlflow.md#backup-steps-are-incomplete) |
| Keycloak | Postgres dump (port 5433) + `*.pass` | [Keycloak](keycloak.md#backup-command-misses-the-port) |
| Model Hub | API key hashes, shares, limit exceptions (Kubernetes); `MODELS_ROOT` | [Model Hub](model-hub.md#no-backup-or-restore-for-the-kubernetes-side) |
| Monitoring | `PROM_DATA` | [Monitoring](monitoring.md#smaller-items) |
| Admin log | `/var/log/aistack-admin.log` (no rotation) | [Admin page](admin-page.md#audit-log-gaps) |

The [Everyday tasks → Backups](../admin/daily.md#backups) table lists what to keep until a backup script exists.

## Smaller items

- `NFS_SERVER` is described as "Dummy: master exports /home" in `site.conf:107`. Is it a placeholder to replace? (Question)
- `REGISTRY_IMAGE=registry:2` is the one image not pinned to an exact version. (Gap) — [Registry](registry.md#registry-image-not-pinned)
- The top-level README's laptop tunnel forwards port 30500 for MLflow, which may no longer be needed with Keycloak.
  (Docs) — [MLflow](mlflow.md#workflowmd-is-out-of-date-for-keycloak)
- A separate login node that is also a Kubernetes node gets no containerd registry config. (Gap) — [Registry](registry.md#uneven-node-coverage-with-a-separate-login-node)
