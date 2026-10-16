# Troubleshooting

Start here. Each component's install page has its own full **Symptom | Cause | Fix** table:
[Registry](install/registry.md#troubleshooting) ·
[LDAP](install/ldap.md#troubleshooting) ·
[Portal](install/portal.md#troubleshooting) ·
[MLflow](install/mlflow.md#troubleshooting) ·
[VS Code](install/vscode.md#troubleshooting) ·
[Model Hub](install/model-hub.md#troubleshooting) ·
[Keycloak](install/keycloak.md#troubleshooting) ·
[Jupyter](install/jupyter.md#troubleshooting) ·
[Admin page](install/admin-page.md#troubleshooting) ·
[Monitoring](install/monitoring.md#troubleshooting).
Problems that come from the code itself are in [Known issues](../known-issues/index.md).

## Top problems

| Symptom | Likely cause | Fix |
|---|---|---|
| Model pod runs but has no GPU inside | slurm-bridge was upgraded and its scheduler settings were reset | `5-model-hub/1-setup.sh`, then redeploy |
| Portal change doesn't show | the portal caches settings per user | Restart Web Server; hard-reload the page |
| Job gets 401 from MLflow | token missing or stale | `3-mlflow/3-sync-tokens.sh` |
| User gets 403 from MLflow on an experiment | someone else owns that experiment name | use `<user>/<name>` |
| Everything `ErrImagePull` after a reboot | registry container not started | rerun `0-registry/setup-registry.sh` |
| Gateway 403 for everyone right after install | user namespaces don't give it read rights yet | `5-model-hub/2-sync-users.sh` |
| Shared with a team, member still gets 403 | the member's namespace doesn't list the team yet | `2-sync-users.sh <member>`, wait a minute |
| Wrong home folder / old password works on one node | a local account with the same name as an LDAP user | `1-ldap/menu.sh` → Check → remove the local copy |
| Admin isn't in the admin's menu | not installed, or the portal caches the app list | `8-admin/2-install-ood-app.sh`, then Restart Web Server as the admin |
| `sudo: aistack-admin: command not found` | `sudo` doesn't search `/usr/local/sbin` | use the full path `/usr/local/sbin/aistack-admin` |
| Admin page: "the admin helper didn't answer" | root program or its sudo rule missing | `8-admin/1-install.sh` |
| A model waits although the GPU is free | a job without `--mem` took the whole node's memory | ask users to set `--mem`; site fix: `DefMemPerCPU` in `slurm.conf` (open question) |
| New pods blocked in every namespace | slurm-bridge's webhook is down | check its pods in namespace `slurm` |
| First portal install stops: `OOD_AUTH=keycloak, but realm … doesn't answer` | `site.conf` ships `keycloak`, but Keycloak isn't installed yet | install the portal with `OOD_AUTH=ldap` first ([order](index.md#install-order)) |
| `Run with sudo.` / `Run this on <host>.` | wrong user or host | see "Runs on" in [Scripts](../reference/scripts.md) |
| `<node>: SKIPPED (unreachable)` / `NOT done on <node>` | root SSH from master to that node failed (5 s timeout) | fix SSH or bring the node up, rerun the script |
| `WARNING: firewalld isn't running … open to the whole network` | firewalld off on that host | start it and rerun, or protect the port another way |
| `… resolve_ctls_from_dns_srv … Unknown host` under `sudo` | `sudo` dropped `SLURM_CONF` and the Slurm path | set `SLURM_BIN`/`SLURM_CONF` in `site.conf`; rerun |
| Password reset fails: `Confidentiality required (13)` | the command used plain `ldap://` | use `1-ldap/set-password.sh` or the ldaps command in [LDAP](install/ldap.md#passwords) |
| User's Model Hub stops working: `Unauthorized` | their Kubernetes certificate (1 year) expired | `5-model-hub/2-sync-users.sh <user>` |
| Alert `CertificateExpiresSoon` | portal, LDAP or gateway certificate ends within 30 days | rerunning the scripts doesn't renew it: see [certificate renewal](../known-issues/install-and-docs.md#certificate-renewal-isnt-really-documented) |
| Jupyter page: `Could not ask Slurm for your sessions` for one user only | an old `~/.jupyter-sessions/<job>.kernels` folder | delete those folders ([known issue](../known-issues/jupyter.md#the-page-can-break-on-a-leftover-kernels-folder)) |

## Rerun rules (easy to forget)

| After you… | Rerun |
|---|---|
| upgrade slurm-bridge (`helm upgrade`) | **`5-model-hub/1-setup.sh`**, or GPUs stop working inside model pods |
| change anything in `site.conf` | the scripts of that component, then Restart Web Server |
| change GPU nodes | `4-vscode/3-install-ood-app.sh`, `7-jupyter/2-install-ood-app.sh`, `5-model-hub/1-setup.sh` |
| add any node | the [new-node checklist](../known-issues/install-and-docs.md#add-a-compute-node-is-incomplete) |
| change `ENDPOINT_HOURS_DEFAULT` | `5-model-hub/7-install-ood-app.sh` |
| change `USER_GPU_QUOTA` or `ENDPOINT_HOURS_MAX` | `5-model-hub/2-sync-users.sh` (all users) |
| deploy MLflow after the portal (Keycloak mode) | `2-ood/setup-ood.sh` (adds MLflow's single-sign-on header) |
| renew the portal's certificate | `6-keycloak/1-install.sh` (so Keycloak trusts it) |
| change a monitoring rule | edit `9-monitoring/rules.yml`, rerun `9-monitoring/1-install.sh` (never edit `/etc/prometheus`) |
| change a portal app | its install script + Restart Web Server |
| change the home page cards | `2-ood/setup-ood.sh` + Restart Web Server |

