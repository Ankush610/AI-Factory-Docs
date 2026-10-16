# 8 · Admin page

> A browser admin page, usable only by `OOD_ADMIN`, that does every action through one small root program on
> master (`aistack-admin`). The page reaches it over a restricted SSH key; on master's terminal you use `sudo`.

**Folder:** [`OOD-Install/8-admin`](https://github.com/Ankush610/OOD-Install/tree/main/8-admin) ·
**Explained in:** [Security and isolation](../../how-it-works/11-security-and-isolation.md#the-admin-page) ·
**Actions:** [Admin helper reference](../../reference/admin-actions.md) ·
**Known issues:** [Admin page](../../known-issues/admin-page.md)

## What it sets up

- **The helper** `ADMIN_HELPER` (`/usr/local/sbin/aistack-admin`) on master, root-owned. It does a fixed list of
  actions, checks every argument, and calls the same scripts as the terminal (`1-ldap/*.sh`,
  `5-model-hub/2-sync-users.sh`, `scancel`, `kubectl`).
- **A sudo rule:** `OOD_ADMIN` may run the helper, and nothing else, without a password.
- **The page's SSH key** `ADMIN_KEY` on the login node, readable only by `OOD_ADMIN`. Master's root accepts it only
  from `LOGIN_IP`, and only to run the helper:
  `restrict,from="<LOGIN_IP>",command="<ADMIN_HELPER> --ssh" ssh-ed25519 … aistack-admin`.
- **The log** `ADMIN_LOG` (root only): every change and every refused attempt.
- **The portal app** **Interactive Apps → Admin** with four tabs: Running, Users, Teams, Log. Only `OOD_ADMIN`'s
  group can read its folder, so nobody else sees it.

## Before you start

| Script | Runs on | Root |
|---|---|---|
| `1-install.sh` | master | yes |
| `2-install-ood-app.sh` | login node | yes |

- A local `OOD_ADMIN` account on master **and** on the login node. Give it a **private primary group** (the app
  folder is readable by that group).
- Root SSH from master to the login node.
- sshd on master accepts root key logins: `PermitRootLogin prohibit-password` (or more).
- Port 22 on master open for `LOGIN_IP`.
- **Done first:** [LDAP](ldap.md), [Portal](portal.md); for the full feature set [Model Hub](model-hub.md) and
  `kubectl` with `/etc/kubernetes/admin.conf`. The page copies Model Hub's look from the repo.
- No internet needed.

## Install

### 1. The helper (master)

```bash
sudo bash 8-admin/1-install.sh
```

| Step | What it does |
|---|---|
| 1. Helper | runs the helper's self-test, fills in the repo path, installs it (only if changed) |
| 2. sudo rule | `/etc/sudoers.d/aistack-admin` (440), checked with `visudo` first |
| 3. Log | creates `ADMIN_LOG`, `root:root 600` |
| 4. SSH key | on the login node: creates `ADMIN_KEY` (ed25519, no passphrase) if missing, owner `OOD_ADMIN`, 600, plus a pinned `known_hosts`. The private key never leaves the login node. On master: replaces the line ending in ` aistack-admin` in root's `authorized_keys` |
| 5. Check | over the key, `teams` works and `id` **fails**; `sudo` works for `OOD_ADMIN`; an LDAP user is refused and can't read the key |

**Expect:** lines like `admin on master, over its key: {"ok": true, …}`, `the key runs nothing but the helper`,
and `<user> (an LDAP user): sudo refused, can't read the key, as it should be`. Any `WRONG:` line stops the script.

### 2. The page (login node)

```bash
sudo bash 8-admin/2-install-ood-app.sh
```

It copies the app (with Model Hub's styles), writes `site.json` (the admin's name and the exact SSH command), sets
`root:<OOD_ADMIN's group>` with 750/640, then checks the admin can read it and reach the helper, and that an LDAP
user **can't** read it.

**Expect:** `passenger_wsgi.py: OK`, `manifest.yml: OK (Interactive Apps)`, `admin can read it: OK`,
`admin reaches the helper on master: OK`, `<user> can't see it: OK`, `Done.`

### 3. Open it

Log in to the portal as `OOD_ADMIN`, click **Restart Web Server**, open **Interactive Apps → Admin** (or the
**Admin Panel** card in the home page's Admin section). Both scripts are safe to rerun; the key is kept.

## Everyday tasks

The page is described in [Everyday tasks → The admin page](../daily.md#the-admin-page). Every button maps to one
helper action; the full list is in the [Admin helper reference](../../reference/admin-actions.md).

From master's terminal, as `OOD_ADMIN`. Use the **full path**: `sudo` on AlmaLinux 9 doesn't search `/usr/local/sbin`.

```bash
sudo /usr/local/sbin/aistack-admin                     # list the actions
sudo /usr/local/sbin/aistack-admin jobs
printf '%s\n' 'Their-1st-pass' | sudo /usr/local/sbin/aistack-admin add-user bob
sudo /usr/local/sbin/aistack-admin limits bob --gpus 2
sudo /usr/local/sbin/aistack-admin cancel-job 1382 'Idle GPU for 6 hours: please use sbatch.'
sudo tail /var/log/aistack-admin.log
```

| Task | How |
|---|---|
| Helper changed in the repo | rerun `1-install.sh` (reinstalls only if different) |
| Page changed | rerun `2-install-ood-app.sh`, then Restart Web Server as `OOD_ADMIN` |
| Changed `OOD_ADMIN`, `LOGIN_IP`, `ADMIN_KEY` or `ADMIN_HELPER` | rerun both scripts (they are written into the sudo rule, the `authorized_keys` line and `site.json`) |
| Master's SSH host keys changed | rerun `1-install.sh` (rewrites `known_hosts`) |
| Rotate the page key | delete `ADMIN_KEY` and `ADMIN_KEY.pub` on the login node, rerun `1-install.sh` |
| Back up | keep `ADMIN_LOG` with the other backups |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `sudo: a password is required` | the sudo rule is missing, or a later rule wins | rerun `1-install.sh` |
| `sudo: aistack-admin: command not found` | `sudo` doesn't search `/usr/local/sbin` | use `/usr/local/sbin/aistack-admin` |
| Page: `the admin helper didn't answer: Permission denied (publickey)` / `Host key verification failed` | key line missing; `LOGIN_IP` isn't the address master sees; `PermitRootLogin no`; or master's host key changed | rerun `1-install.sh`; set `PermitRootLogin prohibit-password`; check `LOGIN_IP` in `/var/log/secure` |
| Page: `the admin helper didn't answer` | helper missing, or the key line is gone | rerun `1-install.sh` |
| `<user> may not use the admin tool` / page 403 `only <admin> may use the admin page` | not `OOD_ADMIN` | log in as `OOD_ADMIN` (refusals are logged) |
| Admin app not in the menu | not installed, the app list is cached, or `manifest.yml` didn't parse | run `2-install-ood-app.sh`, Restart Web Server as the admin |
| Users/Teams/Running tab: `kubectl …` error | Model Hub not set up or Kubernetes down (the Teams tab also loads users) | `kubectl get nodes` as root |
| A change seems to hang | another change holds the lock (e.g. a removal waits for a namespace) | wait; `sudo tail /var/log/aistack-admin.log` |
| Page: `bad request: … timed out after 900 seconds` | a helper call ran too long; it may still finish | check the log, then refresh |
| A user started something over SSH and it's not in Running | only Slurm jobs are listed | `ps -u <user>` |
| User doesn't see "Stopped by the admin" | old VS Code/Jupyter install without notices | rerun `4-vscode/3-install-ood-app.sh` and `7-jupyter/2-install-ood-app.sh` |
| No bell, or the bell's ✕ fails | the bell or `aistack_notices` isn't installed | rerun `2-ood/setup-ood.sh`, Restart Web Server |
| Stop refused: `the reason: …` | no message, too long (> 1000), or control characters | write why; the user reads it |
| `cancel-job`: `… is a Model Hub endpoint: use stop-endpoint` | the job is in the slurm-bridge partition | use **Stop** on the Endpoint row |
| `cancel-job`: `no job N (finished already?)` | the job ended | refresh |
| `<name> is a local admin account: never removed` | removing `KEEP_LOCAL`/`OOD_ADMIN` | intended |
| `No local account <OOD_ADMIN>` | not on master | create it, or fix `site.conf` |
| `visudo refused the rule: nothing changed` | odd `OOD_ADMIN` or `ADMIN_HELPER` value | fix `site.conf`, rerun |
| `No key from <LOGIN_HOST>` | root SSH to the login node failed, or no `OOD_ADMIN` there | fix SSH; create the account |
| `WRONG: the key ran 'id'` | another `authorized_keys` line carries the same key without the forced command | remove it, rerun |
| `WRONG: <user> could run it too` / `can read …ADMIN_KEY` | another sudo rule grants too much, or key permissions are wrong | check `/etc/sudoers.d/*`; `chmod 600`, `chown OOD_ADMIN` the key |
| `WRONG: <user> can read <APP>` | that user shares `OOD_ADMIN`'s primary group | give `OOD_ADMIN` a private group, rerun |
| `No <ADMIN_KEY>: run 1-install.sh on master first.` | wrong order | run step 1 first |
