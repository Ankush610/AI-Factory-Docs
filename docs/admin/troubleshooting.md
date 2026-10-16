# Troubleshooting

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

## Rerun rules (easy to forget)

| After you… | Rerun |
|---|---|
| upgrade slurm-bridge (`helm upgrade`) | **`5-model-hub/1-setup.sh`**, or GPUs stop working inside model pods |
| change anything in `site.conf` | the scripts of that component, then Restart Web Server |
| change GPU nodes | `4-vscode/3-install-ood-app.sh`, `7-jupyter/2-install-ood-app.sh`, `5-model-hub/1-setup.sh` |
| change a portal app | its install script + Restart Web Server |
| change the home page cards | `2-ood/setup-ood.sh` + Restart Web Server |

