# 1 · Users (LDAP + SSSD)

> One user list for the whole cluster: a 389 Directory Server on master, SSSD on every node, and scripts to add,
> remove and manage people, passwords and teams.

**Folder:** [`OOD-Install/1-ldap`](https://github.com/Ankush610/OOD-Install/tree/main/1-ldap) ·
**Explained in:** [Users and login](../../how-it-works/03-users-and-login.md) ·
**Known issues:** [LDAP](../../known-issues/ldap.md)

## What it sets up

- **389 Directory Server** on master, instance `slapd-<LDAP_INSTANCE>`, with `ou=People` and `ou=Groups` under
  `LDAP_BASE`, a self-signed certificate, and two access rules: anyone may read everything except passwords, and
  each user may change their own password.
- **SSSD** on every node (master, login node, compute nodes), so each person has the same user ID everywhere.
- **Admin scripts:** `add-user.sh` (replaces `useradd`), `remove-user.sh`, `set-password.sh`, `group.sh` (teams),
  and `menu.sh`, a text menu over all of them.

One password works for SSH, Slurm, the portal and the MLflow page.

## Before you start

| Script | Runs on | Notes |
|---|---|---|
| `1-server.sh` | master | installs `389-ds-base`, `openldap-clients` |
| `import-local-users.sh` | master | optional |
| `2-client.sh` | **every** node: master, the login node, each compute node | installs `sssd`, `sssd-ldap`, `oddjob-mkhomedir`, `openldap-clients`, so each node needs package access |
| `add-user.sh`, `remove-user.sh`, `set-password.sh`, `group.sh`, `menu.sh` | master | `menu.sh` needs a terminal and `whiptail` (`dnf install newt`) |

All need root. You also need:

- `/home` shared from master to every node. The CA and the OOD-Install folder live there, so compute nodes run
  `2-client.sh` straight from the shared copy.
- `MASTER_HOST` resolving on every node: `LDAP_URI` (`ldaps://master`) must match the certificate name.
- Root SSH from master to `LOGIN_NODES` and `COMPUTE_NODES`, for `add-user.sh`, `remove-user.sh` and the menu's
  Check.

## Settings used

| Variable | Example | Meaning |
|---|---|---|
| `LDAP_BASE` | `dc=aistack,dc=local` | root of the tree. **Can't change after install** |
| `LDAP_INSTANCE` | `aistack` | 389 DS instance (`/etc/dirsrv/slapd-aistack`, unit `dirsrv@aistack`) |
| `LDAP_URI` | `ldaps://$MASTER_HOST` | what clients use (636, TLS) |
| `LDAP_DM`, `LDAP_DM_PASS_FILE` | `cn=Directory Manager`, `/root/.ldap-dm.pass` | the LDAP super-admin and its password file |
| `LDAP_CA` | `/home/apps/ldap/ca.crt` | the server CA on the shared `/home` |
| `MIN_UID`, `MAX_UID` | `1000`, `59999` | user ID range for people |
| `LDAP_LAST_UID_FILE` | `/root/.ldap-last-uid` | highest ID ever handed out |
| `KEEP_LOCAL` | `admin` | local accounts that never go into LDAP |

Full list: [`site.conf` reference](../../reference/site-conf.md#ldap-1-ldap).

## Install

### 1. LDAP server (master)

```bash
sudo bash 1-ldap/1-server.sh
```

| Step | What it does |
|---|---|
| Packages | `389-ds-base`, `openldap-clients` |
| Directory Manager password | generates 32 hex characters into `LDAP_DM_PASS_FILE` if missing, with no trailing newline |
| Instance | `dscreate` with ports 389/636, a self-signed certificate, suffix `LDAP_BASE`. Skipped if it already exists |
| Tree and access rules | `ou=People`, `ou=Groups`, the two access rules |
| CA | copies the instance CA to `LDAP_CA`, mode 644 |
| Firewall | opens services `ldap` and `ldaps` if firewalld runs |
| Check | `dsctl status`, both OUs, an `ldaps` search with the shared CA |

**Expect:** `ou=People: OK`, `ou=Groups: OK`, `ldaps with the shared CA: OK`.

### 2. Import existing local users (optional, master)

Only if master already has people as local accounts. It copies users in `MIN_UID`–`MAX_UID` (except `KEEP_LOCAL`
and `nologin` shells) into LDAP with the **same** user ID and password hash. Existing LDAP entries are never changed.

```bash
sudo bash 1-ldap/import-local-users.sh --dry-run   # review; hashes hidden
sudo bash 1-ldap/import-local-users.sh
```

**Expect:** one `ok <user> <uid>` line per user. Any `WRONG` line stops the script.

### 3. SSSD on every node

Run as root on master, then on the login node (if separate), then on **every** compute node:

```bash
sudo bash 1-ldap/2-client.sh
```

| Step | What it does |
|---|---|
| Packages | `sssd sssd-ldap oddjob-mkhomedir openldap-clients` |
| Trust the CA | copies `LDAP_CA` into the system trust store |
| Clash check | stops if a local account holds an LDAP user's or group's ID under another name |
| SSSD config | backs up an existing `sssd.conf` once, writes a new one, `root:root 0600` |
| Switch | `authselect select sssd with-mkhomedir`, enables `oddjobd`, restarts `sssd` |
| Check | `getent -s sss passwd <first LDAP user>` |

**Expect:** `Done on <host>` and the first LDAP user's line, or `LDAP has no users yet`.

You can also use `menu.sh` → Setup → "A node uses LDAP".

### 4. First people (master)

```bash
sudo bash 1-ldap/add-user.sh <name>
```

Check on any node: `getent passwd <name>`.

## Everyday tasks

All on master, as root. `sudo bash 1-ldap/menu.sh` does every task below from a menu. The
[Admin page](../daily.md#the-admin-page) calls the same scripts.

### Add a person

```bash
sudo bash 1-ldap/add-user.sh <name>                                         # asks for the password
printf '%s\n' "$PW" | sudo bash 1-ldap/add-user.sh <name> --password-stdin   # 8+ characters
```

It refuses a bad name, a name that already exists, or an existing `/home/<name>` (a removed person's kept home).
Then:

1. Picks the next free user ID. IDs are never reused (see [How it works](../../how-it-works/03-users-and-login.md)).
2. Creates the LDAP user and their personal group.
3. Sets the password over ldaps.
4. Creates `/home/<name>` (mode 700).
5. Writes their MLflow job token, if MLflow answers.
6. Gives them a subuid/subgid range on each login node (for rootless podman). Unreachable nodes are reported with
   the command to run later.
7. Creates their Model Hub namespace, if Model Hub is set up.
8. Marks the password one-time in Keycloak, if Keycloak is installed. The user picks their own at the first web login.
9. Checks `getent passwd` and `id`.

### Remove a person

```bash
sudo bash 1-ldap/remove-user.sh <name> [--delete-home] [--delete-mlflow] [--yes]
```

It asks you to type the name unless you pass `--yes`. It refuses `KEEP_LOCAL` and `OOD_ADMIN`.

| Step | What happens |
|---|---|
| 1 | records the user ID so it is never reused |
| 2 | stops everything: Slurm jobs, the portal session, every process on the login and compute nodes |
| 3 | Model Hub: removes them from every share, deletes namespace `u-<name>` and their volumes (files are kept) |
| 4 | MLflow: models only they manage become private (or are deleted with `--delete-mlflow`); with `--delete-mlflow` their experiments are deleted too; the MLflow login is removed |
| 5 | Keycloak: signs them out everywhere |
| 6 | LDAP: removes them from every team, deletes their group and entry; then removes them from Keycloak |
| 7 | stops everything again (an open tab or queued job may have restarted something) |
| 8 | flushes the SSSD cache on every node |
| 9 | deletes the home, only with `--delete-home` and only if it is `/home/<name>` |
| 10 | checks `getent passwd <name>` returns nothing |

The LDAP delete comes last on purpose: until then a failed run can simply be run again.

### Passwords

```bash
# set a one-time password (one line on stdin, 8+ characters)
printf '%s\n' "$PW" | sudo bash 1-ldap/set-password.sh <name>

# by hand: must use ldaps; plain ldap:// fails with "Confidentiality required (13)"
sudo LDAPTLS_CACERT=/home/apps/ldap/ca.crt ldappasswd -x -H ldaps://<MASTER_HOST> \
  -D "cn=Directory Manager" -y /root/.ldap-dm.pass -S uid=<name>,ou=People,<LDAP_BASE>

# test a password
ldapwhoami -x -H ldaps://<MASTER_HOST> -D uid=<name>,ou=People,<LDAP_BASE> -W
```

Users change their own password with `passwd` over SSH, or at the Keycloak login page.

### Teams

Teams are LDAP `groupOfNames` entries used **only** to share Model Hub endpoints. They are not Linux groups and
give no file rights.

```bash
sudo bash 1-ldap/group.sh list [team]
sudo bash 1-ldap/group.sh create <team>
sudo bash 1-ldap/group.sh add <team> <user>...
sudo bash 1-ldap/group.sh remove <team> <user>...
sudo bash 1-ldap/group.sh delete <team>
```

After a change it re-syncs the affected people in Model Hub. The gateway sees it within a minute.

### Other tasks

| Task | How |
|---|---|
| Add a node | as root on the new node: `bash /home/…/OOD-Install/1-ldap/2-client.sh` (see also [daily tasks](../daily.md)) |
| Find local accounts that hide an LDAP user | `menu.sh` → Check. It looks on master, login and compute nodes. Removing one runs `userdel` without `-r` (the home stays) |
| Look someone up | `getent passwd <name>` (local or LDAP), `getent -s sss passwd <name>` (LDAP only) |
| Search LDAP | `ldapsearch -x -H ldap://localhost -b ou=People,<LDAP_BASE> uid=<name>`. Always add `-x` |
| Forget cached users after a change | `sudo sss_cache -E` |
| Undo SSSD on one node | `authselect select local --force; systemctl stop sssd` |
| Back up | the directory **and** `/root/.ldap-last-uid` and `/root/.ldap-dm.pass` (see [known issues](../../known-issues/ldap.md#no-backup-or-restore-procedure)) |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `1-server.sh`: `ldap_bind: Invalid credentials (49)` | the password file has a trailing newline | the script strips it now; rerun |
| `Directory Manager login failed: … does not match the instance password` | the password file was replaced after the instance was created | restore the original `LDAP_DM_PASS_FILE` |
| `ou=People missing` / `ldaps with the shared CA FAILED` | tree add failed, or the CA or hostname doesn't match | check step 4's output; check `MASTER_HOST` resolves and `LDAP_CA` is the instance CA |
| `2-client.sh`: `No <LDAP_CA> … (and is /home mounted?)` | server not set up, or `/home` not mounted | run `1-server.sh` on master; mount `/home` |
| `sssd` fails: `File ownership and permissions check failed` | `sssd.conf` isn't `root:root 0600` | `chown root:root /etc/sssd/sssd.conf; chmod 600 …; systemctl restart sssd` |
| `CLASH: local user 'X' has UID N, which is 'Y' in LDAP` | a local account (often a service like `node_exporter`) holds that ID | on that node: stop `X`, give it a free ID below 1000 (`groupmod`/`usermod`), `chown` its files, start it, rerun `2-client.sh` |
| `SSSD cannot see LDAP users` | CA not trusted, or `MASTER_HOST` doesn't resolve | `LDAPTLS_CACERT=<LDAP_CA> ldapsearch -x -H ldaps://<MASTER_HOST> -b <LDAP_BASE> -s base`; check `/etc/hosts`; `journalctl -u sssd` |
| Slurm job: `unknown userid <uid>` | `2-client.sh` not run on that compute node | run it there; check with `srun -w <node> id <name>` |
| User sees their files owned by a number | their LDAP ID differs from the old local one | set `uidNumber` in LDAP to the old value, then `sss_cache -E` |
| `ldappasswd`: `Confidentiality required (13)` | 389 DS changes passwords only over TLS | use `-H ldaps://<MASTER_HOST>` with `LDAPTLS_CACERT` |
| `ldapwhoami`/`ldapsearch`: `SASL/GSS-SPNEGO … No credentials` | missing `-x`, so it tries Kerberos | add `-x` |
| Imported user can't log in | the local account was locked, so no password was copied | set one with `set-password.sh` |
| `add-user.sh`: `/home/<name> exists` | a removed person's home was kept | move or delete it first |
| `add-user.sh`: `NOT on <node> (unreachable)` | root SSH to a login node failed | run the printed command on that node later |
| `Password too short (8+ characters).` | stdin password too short | use 8 or more characters |
| `remove-user.sh`: `No LDAP user '<name>'` on a rerun | the LDAP delete already ran | `sss_cache -E` on each node; finish the rest by hand ([known issue](../../known-issues/ldap.md#a-failed-removal-cant-be-finished-by-a-rerun)) |
| A removed person still opens the MLflow page for a few minutes | MLflow caches a good login for 5 minutes | wait; their account has no rights left |
| `MLflow not answering …: NOT cleaned` | MLflow was down during removal | later, in MLflow as admin: delete user `<name>` |
| `home is <dir>, not /home/<name>: NOT deleted` | unusual `homeDirectory` in LDAP | check and delete by hand |
| `menu.sh`: `LDAP doesn't answer` / `Needs whiptail` | `dirsrv` down / package missing | `systemctl status dirsrv@<LDAP_INSTANCE>` / `dnf install newt` |
| Wrong home or old local password works on one node | a local account with the same name wins over LDAP | `menu.sh` → Check → remove the local copy |
| `group.sh`: `'<team>' is a user name` / `No team '<team>'` | team name equals a user, or not created | pick another name / `group.sh create <team>` |
| Alert `ServiceDown` for `dirsrv@*` or `sssd` | LDAP or SSSD stopped: nobody can log in | `systemctl start …`, `journalctl -u <unit> -n 50` |
