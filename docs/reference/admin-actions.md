# Admin helper reference

Every action of the root helper `aistack-admin` (installed by [8-admin](../admin/install/admin-page.md)). Each
button on the Admin page is one call of one action.

- **From the page:** the page sends the action over the restricted SSH key; passwords go on standard input.
- **From master's terminal:** `sudo /usr/local/sbin/aistack-admin <action> [args]`, as `OOD_ADMIN`. Pipe passwords
  on standard input: `printf '%s\n' '<password>' | sudo /usr/local/sbin/aistack-admin add-user bob`.
- **Changes** run one at a time (a lock), and are logged to `/var/log/aistack-admin.log` with who, what, when and
  the result. **Refused** attempts are logged too. **Reads** are not logged.
- Passwords are masked as `********` in output and in the log.

## Actions

| Action | Arguments | Stdin | Kind | What it runs | Page |
|---|---|---|---|---|---|
| *(none)* | – | – | – | prints the list of actions | – |
| `users` | – | – | read | LDAP people + teams + each person's Model Hub limits | Users tab; Teams tab (member picker) |
| `teams` | – | – | read | LDAP teams (`groupOfNames`, not personal groups) | Teams tab |
| `endpoints` | – | – | read | every Model Hub endpoint (Services, pods, GPU claims) | terminal only |
| `jobs` | – | – | read | `squeue -a --json`, joined with endpoints | Running tab (every 5 s) |
| `log` | – | – | read | the last 200 lines of the log, newest first | Log tab |
| `add-user` | `NAME` | one-time password | change | `1-ldap/add-user.sh NAME --password-stdin` | Users → **Add user** |
| `remove-user` | `NAME [--delete-home] [--delete-mlflow]` | – | change | `1-ldap/remove-user.sh NAME [flags] --yes`. Refuses `KEEP_LOCAL` and `OOD_ADMIN` | Users → **Remove** (type the name) |
| `set-password` | `NAME` | one-time password | change | `1-ldap/set-password.sh NAME` | Users → **Password** |
| `team-create` | `TEAM` | – | change | `1-ldap/group.sh create TEAM` | Teams → **New team** |
| `team-delete` | `TEAM` | – | change | `1-ldap/group.sh delete TEAM` | Teams → trash icon |
| `team-add` | `TEAM USER…` | – | change | `1-ldap/group.sh add TEAM USER…` | Teams → "Add someone…" |
| `team-remove` | `TEAM USER…` | – | change | `1-ldap/group.sh remove TEAM USER…` | Teams → ✕ on a member |
| `limits` | `USER --gpus N` \| `--max-hours H` \| `--reset` | – | change | `5-model-hub/2-sync-users.sh USER <option>` | Users → **Limits** |
| `stop-endpoint` | `USER NAME REASON` | – | change | marks the endpoint's Service "stopped by the admin" with the reason, deletes its pod (the Slurm job ends, the GPU is freed), writes a notice | Running → **Stop** on an Endpoint row |
| `cancel-job` | `JOBID REASON` | – | change | `scancel JOBID`, writes a notice. Refuses Model Hub endpoint jobs | Running → **Stop** on a Session or Job row |
| `--test` | – | – | – | self-test (no root needed) | – |
| `--ssh` | request in `SSH_ORIGINAL_COMMAND` | – | – | the forced-command entry point used by the page's key | – |

## Argument rules

| Argument | Rule |
|---|---|
| User name | `^[a-z_][a-z0-9_-]{0,31}$` (same as `add-user.sh`) |
| Team name | `^[a-z_][a-z0-9_.-]{0,31}$` (same as `group.sh`) |
| Endpoint name | `^[a-z0-9]([a-z0-9-]{0,40}[a-z0-9])?$` (same as Model Hub) |
| `--gpus` | 0 to 8 |
| `--max-hours` | 1 to 8760 |
| Job ID | 1 to 10 digits |
| Reason | 1 to 1000 characters after trimming; newlines allowed, other control characters refused |

Anything else is refused with `unknown action or wrong arguments` (and logged).

## How the Running tab classifies jobs

| Type | When |
|---|---|
| Endpoint | the job is in the `slurm-bridge` partition, or its name matches a Model Hub endpoint |
| Session | the job's command is under `/var/www/ood/apps/sys/<app>/`: VS Code, Jupyter (or Desktop) |
| Job | anything else (training jobs from `sbatch`) |

## Notices to users

When the admin stops something, the helper writes a file to `ADMIN_NOTICES` (`/home/apps/admin-notices`):

- named `<jobname>.<jobid>` for jobs and sessions (`vscode.1382`), `endpoint.<user>.<name>` for endpoints;
- containing `admin <YYYY-MM-DD HH:MM>` and the reason;
- owned by the user, mode 600, created safely (no symlink tricks) in the sticky 1777 folder.

Users see it under the portal's **bell**, on the VS Code and Jupyter pages ("Stopped by the admin"), and in Model
Hub (an envelope icon on the stopped endpoint). They clear it themselves. A Model Hub redeploy clears the endpoint's
marks.
