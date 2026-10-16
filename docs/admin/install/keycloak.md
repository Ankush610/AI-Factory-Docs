# 6 · Keycloak (single sign-on)

> One login page for every web tool. The portal sends the browser to Keycloak; MLflow and Model Hub ride on the
> portal's session. LDAP stays the only user list and checks every password.

**Folder:** [`OOD-Install/6-keycloak`](https://github.com/Ankush610/OOD-Install/tree/main/6-keycloak) ·
**Explained in:** [Users and login](../../how-it-works/03-users-and-login.md) ·
**Known issues:** [Keycloak](../../known-issues/keycloak.md)

## What it sets up

- **Keycloak 26.8.0 + Postgres 17.11** on master, as two podman containers run by systemd (quadlet units
  `keycloak` and `keycloak-db`), both as the local `keycloak` account (949). Not in Kubernetes, so login keeps
  working when Kubernetes has a problem.
- **Realm `KEYCLOAK_REALM`** with users and groups read from LDAP. Keycloak writes **only passwords** back to LDAP,
  through its own LDAP account `cn=keycloak-passwords`, which may change `userPassword` and nothing else.
- **One OIDC client, `ood`**, for the portal. Its secret is pushed to the login node.
- **`OOD_ADMIN` as a local Keycloak user**, so the admin can still log in when LDAP is down.
- **The login page look** (theme `ai-factory`): the background image and a heading with `PORTAL_TITLE`.

Installing Keycloak doesn't change the portal's login. The portal switches only when you set
`OOD_AUTH=keycloak` and rerun `2-ood/setup-ood.sh`, and you can switch back at any time.

## Before you start

| | |
|---|---|
| **Runs on** | master, as root (both scripts check the host) |
| **Done first** | [Registry](registry.md) (answers on `/v2/`), [LDAP](ldap.md) (answers on `ldap://localhost`), [Portal](portal.md) on the login node (Keycloak needs its certificate) |
| **Needs** | podman ≥ 4.4 with quadlet support; root SSH from master to the login node |
| **Free ports** (first install) | `KEYCLOAK_PORT` (8180), `KEYCLOAK_MGMT_PORT` (9180), `KEYCLOAK_DB_PORT` (5433) |
| **Internet** (first install only) | pull `postgres:<tag>` and build from `quay.io/keycloak/keycloak`. Both are pushed to the local registry |

## Settings used

| Variable | Example | Meaning |
|---|---|---|
| `KEYCLOAK_VERSION` | `26.8.0` | Keycloak release. Never `latest` |
| `KEYCLOAK_IMAGE`, `KEYCLOAK_DB_IMAGE` | `$REGISTRY/keycloak:…-rudra`, `$REGISTRY/postgres:17.11` | the two images in the local registry |
| `KEYCLOAK_USER`, `KEYCLOAK_UID` | `keycloak`, `949` | the account that runs both containers |
| `KEYCLOAK_DATA` | `/home/apps/keycloak` | Postgres data, `*.pass` files, theme (mode 700) |
| `KEYCLOAK_PATH` | `/auth` | URL path under the portal's address (built into the image) |
| `KEYCLOAK_PORT` | `8180` | HTTP on `MASTER_IP`; firewalld lets in only `LOGIN_IP` |
| `KEYCLOAK_MGMT_PORT`, `KEYCLOAK_DB_PORT` | `9180`, `5433` | health/metrics and Postgres, 127.0.0.1 only |
| `KEYCLOAK_URL` | `http://$MASTER_IP:$KEYCLOAK_PORT$KEYCLOAK_PATH` | what Apache and the scripts call |
| `KEYCLOAK_REALM` | `$CLUSTER_ID` | the realm users log in to |
| `KEYCLOAK_ADMIN` | `kcadmin` | Keycloak's own admin; password in `KEYCLOAK_DATA/admin.pass` |
| `OOD_SERVERNAME` | `<portal>` | the public address (`KC_HOSTNAME`) and the client's redirect URL |
| `OOD_OIDC_SECRET` | `/etc/ood/oidc-client.secret` | where the client secret goes on the login node |

## Install

### 1. Portal first, with `OOD_AUTH=ldap` (login node)

```bash
sudo bash 2-ood/setup-ood.sh
```

This makes the portal's certificate and the `/auth` proxy. Until Keycloak runs, `/auth` answers 503.

### 2. Keycloak and Postgres (master)

```bash
sudo bash 6-keycloak/1-install.sh
```

| Step | What it does |
|---|---|
| 1. Checks | podman quadlet, the registry, LDAP on localhost, `OOD_CERT` readable on the login node; free ports if not running yet |
| 2. Account | creates `keycloak` (949) and `KEYCLOAK_DATA`, `KEYCLOAK_DATA/db` (mode 700) |
| 3. Passwords | `admin.pass`, `bootstrap.pass`, `db.pass` once (root, 0600); env files in `/etc/keycloak/` (0600) |
| 4. Images | if not in the registry yet: mirrors Postgres, builds the optimized Keycloak image, pushes both |
| 5. Units | copies the theme, saves master's copy of the portal certificate, writes the two quadlet units, starts or restarts them |
| 6. Firewall | rich rule: `KEYCLOAK_PORT` from `LOGIN_IP` only. Warns if firewalld isn't running |
| 7. Wait | up to 3 minutes for `/health/ready` |
| 8. Admin | creates the permanent admin `kcadmin`, deletes the bootstrap admin |
| 9. Check | unit states, listening ports, and the realm through the portal's Apache |

**Expect:**

```text
units: active active
listening (expect <master-ip>:8180, the other two on 127.0.0.1): ...
through OOD's Apache on master: https://<portal>/auth  ok
```

If the last line shows `HTTP <code>`, rerun `2-ood/setup-ood.sh` on the login node.

### 3. Realm, LDAP link, portal client, local admin (master)

```bash
sudo bash 6-keycloak/2-realm.sh     # asks once for OOD_ADMIN's Keycloak password (8+ characters)
```

| Step | What it does |
|---|---|
| 0. LDAP account | creates `cn=keycloak-passwords` and its access rule (write `userPassword` under `ou=People` only) |
| 1. Realm + LDAP | creates or updates the realm, the LDAP provider (read-only names and groups, writable passwords), then syncs all users and groups |
| 2. Client + admin | creates the `ood` client, pushes its secret to `OOD_OIDC_SECRET` on the login node, creates `OOD_ADMIN` as a local user |
| 3. Check | proves the LDAP account **can't** change another attribute (or stops with `WRONG: …`); prints the user count |

**Check a login** from the login node, as a normal user (the password is read from the terminal):

```bash
read -rsp 'password: ' P; echo; printf '%s' "$P" | curl -sk --data-urlencode grant_type=password \
  --data-urlencode client_id=admin-cli --data-urlencode username=$USER --data-urlencode password@- \
  https://localhost/auth/realms/<realm>/protocol/openid-connect/token | head -c 60; echo; unset P
```

Expect `{"access_token":…`. A wrong password gives `Invalid user credentials`. In a browser:
`https://<portal>/auth/realms/<realm>/account`.

### 4. Switch the portal to Keycloak (login node)

Set `OOD_AUTH=keycloak` in `site.conf`, then:

```bash
sudo bash 2-ood/setup-ood.sh
```

**Expect** the check line `/pun/sys/dashboard -> https://<portal>/auth/realms/<realm>/protocol/openid-connect/auth`.
Open `https://<portal>`: the Keycloak login comes first, then the portal.

For MLflow single sign-on, `3-mlflow/2-deploy.sh` must have run before this, and `3-mlflow/4-install-ood-app.sh`
after it.

**Going back:** `OOD_AUTH=ldap`, rerun `setup-ood.sh`. Nothing in Keycloak needs undoing.

## Everyday tasks

On master, as root.

| Task | How |
|---|---|
| Status / logs | `systemctl status keycloak keycloak-db` · `journalctl -u keycloak -f` |
| Restart | `systemctl restart keycloak-db keycloak` |
| Health | `curl -s http://127.0.0.1:9180/auth/health/ready` |
| Add a person | `1-ldap/add-user.sh <name>`: the password you give is one-time; their first web login asks for their own |
| Forgotten password | `1-ldap/menu.sh` → Set a new password, or the Admin page. One-time again |
| User changes own password | `https://<portal>/auth/realms/<realm>/account`, or `passwd` over SSH |
| Remove a person | `1-ldap/remove-user.sh <name>`: signs them out, deletes them in LDAP, then in Keycloak (in that order) |
| Sign one user out everywhere | `sudo bash -c 'source site.conf; KC_URL=$KEYCLOAK_URL KEYCLOAK_ADMIN=$KEYCLOAK_ADMIN KEYCLOAK_DATA=$KEYCLOAK_DATA KEYCLOAK_REALM=$KEYCLOAK_REALM python3 6-keycloak/kc.py logout <name>'` |
| Admin console | `https://<portal>/auth/admin`, user `kcadmin`, password: `sudo cat /home/apps/keycloak/admin.pass` |
| Change `OOD_ADMIN`'s password | admin console → realm → Users → `<OOD_ADMIN>` → Credentials |
| Re-sync LDAP now | rerun `2-realm.sh` (automatic: changed users every 15 min, full sync daily) |
| Change the login look | edit `theme/ai-factory/…`, rerun `1-install.sh` |
| Portal certificate renewed | rerun `1-install.sh` (fetches the new certificate, so back-channel logout keeps working) |
| `OOD_SERVERNAME` changed | rerun `1-install.sh`, `2-realm.sh` and `2-ood/setup-ood.sh` |
| Upgrade Keycloak | change `KEYCLOAK_VERSION`, rerun `1-install.sh`. Keycloak migrates its database on start |
| Upgrade Postgres (major) | dump and restore; a tag change alone isn't enough |
| Back up | the Postgres database and `KEYCLOAK_DATA/*.pass` (see below) |

### Back up

```bash
# note -p 5433: Postgres doesn't listen on the default port here
podman exec keycloak-db pg_dump -p 5433 -U keycloak keycloak > keycloak.sql
cp -a /home/apps/keycloak/*.pass /etc/keycloak/*.env /safe/place/
```

The folder README's command has no `-p` (see [known issues](../../known-issues/keycloak.md#backup-command-misses-the-port)).
The dump also contains the LDAP account's password, so store it like a secret.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Port N is taken by another program` | something else listens there | pick free ports in `site.conf` |
| `podman has no quadlet support (needs podman >= 4.4)` | old podman | upgrade podman on master |
| `Registry … doesn't answer: run ../0-registry first.` | registry down | `0-registry/setup-registry.sh` |
| `No LDAP on localhost` | 389 DS not running | `systemctl status dirsrv@<instance>` |
| `No OOD certificate … on <LOGIN_HOST>` | portal not installed, or no root SSH | run `2-ood/setup-ood.sh` there; `ssh root@<login-node> true` |
| `UID 949 already belongs to '<x>'` | `KEYCLOAK_UID` taken | pick a free ID below `MIN_UID` |
| `Keycloak not ready after 3 min` | container fails to start | `journalctl -u keycloak -u keycloak-db -n 50` |
| Step 9: `through OOD's Apache …: HTTP <code>` | no `/auth` proxy on the login node, or firewall | rerun `2-ood/setup-ood.sh` there |
| `WARNING: firewalld isn't running` | firewalld stopped | start it and rerun, or block the port another way |
| `/auth/…` gives 503 | Keycloak down, or Apache proxies to an old address, or firewall blocks `LOGIN_IP` | `systemctl status keycloak`; rerun `1-install.sh` and `2-ood/setup-ood.sh`; `curl http://<master-ip>:8180/auth/realms/master` from the login node |
| `/auth` gives 404 | Apache has no `/auth` proxy yet | rerun `2-ood/setup-ood.sh` |
| "HTTPS required" | opened over http, or Apache doesn't send `X-Forwarded-Proto` | use `https://`; rerun `2-ood/setup-ood.sh` |
| Links or redirects point to the wrong host / `Invalid parameter: redirect_uri` | the browser uses a name other than `OOD_SERVERNAME` | set `OOD_SERVERNAME` to what users type; rerun `setup-ood.sh` and `1-install.sh` |
| An LDAP user gets "Invalid user credentials" | wrong password, or not under `ou=People` | test the same password with `ssh`; rerun `2-realm.sh` |
| `keycloak-db` restarts, `Permission denied` | `KEYCLOAK_DATA/db` not owned by 949 | rerun `1-install.sh` |
| Portal: `Error: user … does not exist` after login | that Keycloak user isn't a Linux user (only LDAP users and `OOD_ADMIN` can use the portal) | log in as an LDAP user |
| Portal: login loop or `state` error | old cookies, or the OIDC passphrase changed | clear the site's cookies; keep `/etc/ood/oidc-crypto.passphrase` |
| Portal login broken and you need in now | — | `OOD_AUTH=ldap`, rerun `2-ood/setup-ood.sh` |
| A signed-out user's portal tab keeps working | Keycloak can't reach `https://<portal>/oidc` (certificate not trusted, or the name doesn't resolve on master) | `journalctl -u keycloak \| grep -i backchannel`; after renewing the portal certificate, rerun `1-install.sh` |
| `Keycloak doesn't trust LDAP's CA yet` | installed before passwords were writable | rerun `1-install.sh`, then `2-realm.sh` |
| New password refused / "Could not modify attribute" | Keycloak can't write to LDAP | rerun `2-realm.sh`; `journalctl -u keycloak \| grep -i ldap` |
| Deleting a user in Keycloak fails | Keycloak passes deletes to LDAP, which its account may not do | delete in LDAP first (`remove-user.sh` does this) |
| `Neither the admin nor the bootstrap admin can log in` / `kcadmin can't log in` | the database has another admin password | put it in `KEYCLOAK_DATA/admin.pass`, rerun `1-install.sh` |
| `some LDAP users failed to sync` | LDAP entries Keycloak can't import | `journalctl -u keycloak` |
| `WRONG: cn=keycloak-passwords… could change 'description'` | another access rule on `ou=People` grants too much | check the access rules on `ou=People` |
| `must-change`: `user <n>: not found` | not in LDAP, or provider not set | check LDAP; rerun `2-realm.sh` |
| `remove-user`: `still in LDAP. Delete them there first` | called before the LDAP delete | use `1-ldap/remove-user.sh` |
