# Everyday tasks

## The admin page

Portal → **Interactive Apps → Admin**, or the Admin card on the home page. Only the admin account (`OOD_ADMIN` in
`site.conf`) sees it. Every button calls the same scripts as the terminal, and every action is logged.

| Tab | Shows | Does |
|---|---|---|
| **Running** | everything in Slurm: VS Code / Jupyter sessions, training jobs, Model Hub endpoints; user, node, GPU, since when, ends when, why it waits; GPUs in use per node. Refreshes every 5 s | **Stop** one. The user sees "Stopped by the admin" |
| **Users** | name, user ID, teams, GPU limit, longest run | **Add** (one-time password), **Password**, **Limits**, **Remove** (type the name to confirm; two delete options) |
| **Teams** | each team and its members | new team, add / remove a member, delete a team |
| **Log** | every action, newest first; refused attempts in red | |

The same actions from a terminal (as the admin account): `sudo /usr/local/sbin/aistack-admin <action>`. Run it with
no action for the list. Use the full path: `sudo` doesn't search `/usr/local/sbin` on AlmaLinux 9. Every action and
its rules: [Admin helper reference](../reference/admin-actions.md).

When you **Stop** something, you must write a reason. The user sees it under the portal's bell, on their VS Code or
Jupyter page, or in Model Hub.

Not shown in Running: programs a user started over SSH outside Slurm (`ps -u <user>` by hand).

## Common tasks

| Task | How |
|---|---|
| Who is using which GPU, since when | Admin page → **Running** |
| Stop a session, training job or endpoint | Admin page → Running → **Stop** |
| Add a person | Admin page → Users → **Add**, or `1-ldap/add-user.sh <name>`. Makes everything they need; they choose their own password at first login |
| Remove a person | Admin page → Users → **Remove**, or `1-ldap/remove-user.sh <name> [--delete-home] [--delete-mlflow]` |
| Reset a password | Admin page → Users → **Password** (one-time), or `1-ldap/menu.sh` |
| Manage teams | Admin page → **Teams**, or `1-ldap/menu.sh` |
| Check nodes for clashing local accounts | `1-ldap/menu.sh` → Check |
| Give one person 2 GPUs | Admin page → Users → **Limits**, or `5-model-hub/2-sync-users.sh <user> --gpus 2` |
| Let one person run models longer | Admin page → Users → **Limits**, or `5-model-hub/2-sync-users.sh <user> --max-hours 240` |
| Back to normal limits / see everyone's limits | `--reset` / `--show` |
| Change limits for everyone | edit `site.conf`, rerun `5-model-hub/2-sync-users.sh` |
| Add a training image | copy the `.sif` to `/home/apps/containers/training/<family>/` |
| Add a compute node | several steps: see the [checklist](../known-issues/install-and-docs.md#add-a-compute-node-is-incomplete) |
| Add base models (LLMs) | copy to `/home/apps/models/base/<org>/<model>`, `model-register` it as yourself, **Make public** in Model Hub |
| Renew users' Kubernetes logins (1 year) | rerun `5-model-hub/2-sync-users.sh` (renews those with < 30 days left); do it monthly |
| Change a `site.conf` value | rerun the scripts of the components that use it ([`site.conf` reference](../reference/site-conf.md)), then Restart Web Server |
| Switch the portal login (Keycloak ↔ LDAP) | change `OOD_AUTH`, rerun `2-ood/setup-ood.sh` on the login node |
| Keycloak admin console | `https://<portal>/auth/admin`, user `kcadmin`, password in `/home/apps/keycloak/admin.pass` |
| MLflow admin login | user `admin`, password in `/home/apps/mlflow-shared/admin.pass` |

## Where to look

| What | Where |
|---|---|
| MLflow log | `kubectl -n mlflow logs deploy/mlflow` |
| Keycloak | `systemctl status keycloak keycloak-db`, `journalctl -u keycloak` |
| Public-models job | `journalctl -u model-hub-public` |
| A user's endpoint | Model Hub → endpoint → Logs; or `kubectl -n u-<user> get pods` |
| A user's editor session | `~/.vscode-sessions/<job>.log`, `~/.jupyter-sessions/<job>.log` |
| GPU queue | `squeue -p slurm-bridge`, `squeue` (or Admin page → Running) |
| Gateway calls and key checks | `kubectl -n model-hub logs deploy/gateway -c nginx` / `-c auth` (one JSON line each) |
| Monitoring | `systemctl status prometheus`; firing alerts at `<PROM_URL>/alerts` or in the [Cluster monitor](cluster-monitor.md) |
| What was done on the Admin page (refused attempts too) | Admin page → Log, or `sudo tail /var/log/aistack-admin.log` |

## Backups

| What | Where | Note |
|---|---|---|
| Settings | `OOD-Install/site.conf` | small, keep it with the scripts |
| Users | LDAP (389 DS) + `/root/.ldap-last-uid` + `/root/.ldap-dm.pass` | export the directory; the first file keeps user IDs from being reused |
| Admin actions | `/var/log/aistack-admin.log` | who did what, when |
| Login settings | Keycloak's Postgres + `/home/apps/keycloak/*.pass` | `podman exec keycloak-db pg_dump -p 5433 -U keycloak keycloak`; the dump holds a password |
| Runs and models | `/home/apps/mlflow-shared` as **one piece**, incl. `admin.pass`, `proxy.secret` | losing it loses every run and model; stop the pod for a consistent copy |
| Model Hub state | API key hashes, shares, limit exceptions (in Kubernetes) | not backed up: after a rebuild, users make new keys and re-share |
| Images | the registry storage folder | |
| Model files | `/home/apps/models`, users' `~/models` | part of the `/home` backup |

A single backup and restore script for all of these is roadmap step 9. Until then, see
[Known issues → No backup or restore procedure](../known-issues/install-and-docs.md#no-backup-or-restore-procedure).
