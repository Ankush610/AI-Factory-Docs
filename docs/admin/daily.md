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
no action for the list. Use the full path: `sudo` doesn't search `/usr/local/sbin` on AlmaLinux 9.

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
| Add a compute node | `1-ldap/2-client.sh` on it, rerun `0-registry/setup-registry.sh` |
| Keycloak admin console | `https://<portal>/auth/admin` |

## Where to look

| What | Where |
|---|---|
| MLflow log | `kubectl -n mlflow logs deploy/mlflow` |
| Keycloak | `systemctl status keycloak keycloak-db`, `journalctl -u keycloak` |
| Public-models job | `journalctl -u model-hub-public` |
| A user's endpoint | Model Hub → endpoint → Logs; or `kubectl -n u-<user> get pods` |
| A user's editor session | `~/.vscode-sessions/<job>.log`, `~/.jupyter-sessions/<job>.log` |
| GPU queue | `squeue -p slurm-bridge`, `squeue` (or Admin page → Running) |
| What was done on the Admin page (refused attempts too) | Admin page → Log, or `sudo tail /var/log/aistack-admin.log` |

## Backups

| What | Where | Note |
|---|---|---|
| Settings | `OOD-Install/site.conf` | small, keep it with the scripts |
| Users | LDAP (389 DS) + `/root/.ldap-last-uid` | export the directory; the file keeps user IDs from being reused |
| Admin actions | `/var/log/aistack-admin.log` | who did what, when |
| Login settings | Keycloak's Postgres (`pg_dump`) + its password files | |
| Runs and models | the MLflow data folder, as **one piece** | losing it loses every run and model |
| Images | the registry storage folder | |
| Model files | `/home/apps/models`, users' `~/models` | part of the `/home` backup |

A single backup and restore script for all of these is roadmap step 9.
