# 2 · Web portal (Open OnDemand)

> Installs Open OnDemand 4.2 on the login node, sets its web login (LDAP or Keycloak), connects it to Slurm, and
> gives it the AI-Factory look: the tool-card home page, branding, and the notifications bell.

**Folder:** [`OOD-Install/2-ood`](https://github.com/Ankush610/OOD-Install/tree/main/2-ood) ·
**Explained in:** [Web portal](../../how-it-works/04-web-portal-ood.md) ·
**Known issues:** [Portal](../../known-issues/portal.md)

## What it sets up

- Open OnDemand with Apache on port 443, a self-signed certificate, and the `/node`, `/rnode` and `/auth` proxies.
- The web login, chosen by `OOD_AUTH`:
    - `ldap`: Apache asks for the LDAP password itself (HTTP Basic). `OOD_ADMIN` has a local htpasswd fallback.
    - `keycloak`: Apache sends the browser to Keycloak (OpenID Connect). One login covers the portal, MLflow and Model Hub.
- A Slurm cluster file, so the portal can submit jobs and open a shell.
- The look, through Open OnDemand's own settings only, so an upgrade keeps it: title, logo, colours, the home page
  cards, a shared top bar for our apps, and a **bell** that lists the admin's "your job was stopped" messages.
- A small app, `aistack_notices`, that lets users clear those messages.

## Before you start

| | |
|---|---|
| **Runs on** | the login node (`LOGIN_HOST`), as root |
| **Done first** | `1-ldap/1-server.sh` on master and `1-ldap/2-client.sh` on the login node. The script checks LDAP answers over ldaps |
| **For `OOD_AUTH=keycloak`** | `6-keycloak/1-install.sh` and `2-realm.sh` on master. But Keycloak needs the portal first, so see the order below |
| **Needs** | `OOD_ADMIN` as a local Linux user on the login node; `sbatch` in `SLURM_BIN`; `SLURM_CONF` readable |
| **Internet on the login node** | CRB and EPEL repos, the `ruby:3.3` and `nodejs:22` modules, the Open OnDemand release RPM |
| **Network** | the login node must reach master's 636 (LDAPS) and, for Keycloak, `KEYCLOAK_PORT` |

## Settings used

| Variable | Example | Meaning |
|---|---|---|
| `OOD_VERSION` | `4.2` | Open OnDemand release |
| `OOD_SERVERNAME` | `<portal>` | the exact host users type: certificate name, server name, logout redirect |
| `OOD_AUTH` | `ldap` → `keycloak` | the web login mode |
| `OOD_ADMIN` | `admin` | local web login that works when LDAP is down |
| `OOD_CERT`, `OOD_OIDC_SECRET` | `/etc/pki/tls/certs/ood.crt`, `/etc/ood/oidc-client.secret` | portal certificate; Keycloak client secret |
| `CLUSTER_ID`, `CLUSTER_TITLE` | `aistack`, `AI-Stack` | cluster file name and menu name |
| `PORTAL_TITLE`, `PORTAL_COLOR` | `AI-HPC-Cluster`, `#05295d` | tab title and top bar colour |
| `SLURM_BIN`, `SLURM_CONF` | `/usr/local/bin`, `/etc/slurm/slurm.conf` | where Slurm is |
| `MLFLOW_PREFIX`, `MLFLOW_PROXY_SECRET` | see MLflow | the MLflow single-sign-on header |
| `ADMIN_NOTICES` | `/home/apps/admin-notices` | the folder the bell reads |

## Install

!!! warning "Start with `OOD_AUTH=ldap`"
    On a fresh cluster Keycloak isn't installed yet, and the Keycloak check would stop the script. Set
    `OOD_AUTH=ldap` in `site.conf` for the first run.

### 1. Run the installer (login node)

```bash
sudo bash 2-ood/setup-ood.sh
```

In `ldap` mode it asks once for `OOD_ADMIN`'s web password.

| Step | What happens |
|---|---|
| Checks | root, the right host, LDAP answers over ldaps; in `keycloak` mode also the realm and `OOD_OIDC_SECRET` |
| `== 1. Repos` | CRB, EPEL, `ruby:3.3`, `nodejs:22`, the OSC GPG key, the release RPM (if missing) |
| `== 2.` | creates group `saslauth` if GID 76 is taken, so `cyrus-sasl` installs |
| `== 3. Packages` | `ondemand mod_ssl mod_ldap httpd-tools mod_auth_openidc` |
| `== 4.` | self-signed certificate (RSA 2048, 365 days, CN = `OOD_SERVERNAME`), only if `OOD_CERT` is missing |
| `== 5.` | `ldap`: htpasswd for `OOD_ADMIN` if missing. `keycloak`: `OOD_ADMIN` is a Keycloak local user |
| `== 6. Portal config` | backs up `ood_portal.yml` once, writes the LDAP CA config and the new `ood_portal.yml` (mode 600), runs `update_ood_portal` |
| `== 7. Cluster config` | `clusters.d/<CLUSTER_ID>.yml` (Slurm adapter) |
| `== 8. Look` | branding, `app-bar.css`, home page widget, bell, the `aistack_notices` app; writes `ondemand.d/ai-factory.yml` |
| `== 9.` | `apachectl configtest` (stops if broken, without restarting Apache), restarts `httpd`, opens `https` in firewalld |
| `== 10. Check` | prints HTTP codes and what to expect |

**Expect:**

- `/` returns `302`.
- `ldap` mode: `/pun/sys/dashboard` returns `401`.
- `keycloak` mode: `/pun/sys/dashboard` redirects to `https://<portal>/auth/realms/<realm>/protocol/openid-connect/auth`.
- `/auth/realms/master` returns `200` once Keycloak is installed, `503` before.

### 2. Check a login

```bash
# on the login node, ldap mode
curl -sk -o /dev/null -w '%{http_code}' -u <user> https://localhost/pun/sys/dashboard   # expect 200
```

In the browser, **Clusters → Shell** opens.

### 3. Switch to Keycloak

After [Keycloak](keycloak.md) is installed on master: set `OOD_AUTH=keycloak` in `site.conf`, rerun
`setup-ood.sh` on the login node. Expect the Keycloak login page first, then the portal. **Log Out** in the portal
also ends the Keycloak session.

### 4. After MLflow is deployed

Rerun `setup-ood.sh` once more in `keycloak` mode so Apache adds the MLflow single-sign-on header.
`3-mlflow/2-deploy.sh` reminds you if it is missing.

After every rerun: **Restart Web Server** in the portal's top-right menu.

## Everyday tasks

| Task | How |
|---|---|
| Apply a config change for users | portal → top-right menu → **Restart Web Server** (each user) |
| Apply a `site.conf` change | rerun `setup-ood.sh` on the login node, then Restart Web Server |
| Switch login `ldap` ↔ `keycloak` | change `OOD_AUTH`, rerun `setup-ood.sh`. Nothing in Keycloak needs undoing |
| Keycloak login broken (emergency) | `OOD_AUTH=ldap`, rerun `setup-ood.sh` |
| Rename the cluster | change `CLUSTER_ID`/`CLUSTER_TITLE`, delete the old `/etc/ood/config/clusters.d/<old>.yml`, rerun, Restart Web Server |
| Add a home page card | add `[app folder, title, icon, colour, subtitle, text]` to the list at the top of `2-ood/branding/_ai_tools.html.erb`, rerun, Restart Web Server |
| Change `OOD_ADMIN`'s password | `ldap` mode: `sudo htpasswd /etc/ood/htpasswd <OOD_ADMIN>`. `keycloak` mode: it is a Keycloak local user |
| See a user's admin notices | `ls -l /home/apps/admin-notices` (files are owned by the user) |
| Renew the certificate | see [known issues](../../known-issues/portal.md#certificate-renewal-isnt-documented) |

Check the home page widget without a browser (on the login node, as any user):

```bash
cd /var/www/ood/apps/sys/dashboard
source /opt/ood/ondemand/enable
SECRET_KEY_BASE=x RAILS_ENV=production bin/rails runner 'puts ApplicationController.render(partial: "widgets/ai_tools")'
```

For the bell, use `partial: "layouts/nav/develop_dropdown"`.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Run this on <LOGIN_HOST>` | the portal lives on the login node | run it there, or fix `LOGIN_HOST` |
| `No answer from <LDAP_URI> …` | LDAP server or client not set up, or ldaps/CA failing | `1-ldap/1-server.sh` on master, `1-ldap/2-client.sh` on the login node |
| First install with `OOD_AUTH=keycloak` stops at the realm check | Keycloak not installed yet; it needs the portal first | install with `ldap`, install Keycloak, then switch |
| `No /etc/ood/oidc-client.secret` or `realm … doesn't answer` | `2-realm.sh` not run, or master's firewall blocks `LOGIN_IP` | on master: `6-keycloak/1-install.sh` and `2-realm.sh`; test `curl http://<master-ip>:8180/auth/realms/master` |
| `OOD_AUTH must be ldap or keycloak` | typo in `site.conf` | fix it |
| `id: '<user>': no such user` after `== 5.` | `OOD_ADMIN` isn't a local user | create it, rerun |
| `groupadd: GID '76' already exists` | another group owns GID 76 | the script handles it; by hand: `groupadd -r saslauth; dnf install -y cyrus-sasl` |
| Release RPM 404 | wrong version URL | take the current URL from the official install guide; set `OOD_VERSION` |
| `Apache config error … web server NOT restarted.` | `apachectl configtest` failed | fix the reported error, rerun |
| Login refuses the right password (`ldap` mode) | the browser resends an old login, or the user isn't in LDAP | close every browser window; test with `curl -u` and `ldapsearch` |
| Every LDAP login refused; `Can't contact LDAP server` in Apache's log | the login node can't reach 636, or doesn't trust `LDAP_CA` | `LDAPTLS_CACERT=<LDAP_CA> ldapsearch -x -H ldaps://<MASTER_HOST> -b <LDAP_BASE> -s base`; open 636 for `LOGIN_IP`; rerun |
| Only one user per browser, even in private windows | Basic auth keeps one login per site | use another browser profile; real fix: `OOD_AUTH=keycloak` |
| Browser "site can't be reached" through a tunnel | tunnel on a port other than 443 | tunnel with `-L 443:localhost:443` |
| `resolve_ctls_from_dns_srv … Unknown host` | the portal doesn't see `SLURM_CONF` | set `SLURM_CONF`, rerun, Restart Web Server |
| `Problem talking to database … 'cluster' can't be reached` | a `cluster:` line in the cluster file needs `slurmdbd` | don't add one (the script doesn't) |
| `sbatch: command not found` | wrong `SLURM_BIN` | the folder of `which sbatch` |
| Old logo, footer or settings still shown | the user's portal started before the change | Restart Web Server; hard-reload (Ctrl+Shift+R) |
| Red "widget error" box instead of cards | a mistake in `_ai_tools.html.erb` | fix, rerun, Restart Web Server; test with the `rails runner` command |
| A tool has no card | its folder isn't in the card list, or the user can't open it | add it; `ls -ld /var/www/ood/apps/sys/<app>` |
| Bell missing, or Develop menu gone | the bell replaces the Develop menu by design | rerun `setup-ood.sh`, Restart Web Server |
| Bell ✕ / Clear all: `Could not clear it: HTTP 404` | `aistack_notices` has no nginx config yet | rerun `setup-ood.sh`; quick fix: open `/pun/sys/aistack_notices/` once and press Initialize |
| `MLflow SSO skipped: no /etc/ood/mlflow-proxy.secret yet` | `3-mlflow/2-deploy.sh` not run, or its copy failed | rerun `2-deploy.sh` on master, then `setup-ood.sh` |
| `MLflow SSO skipped: MLFLOW_PREFIX … is not under /node/` | `MLFLOW_PREFIX` empty or elsewhere | set it under `/node/` |
| MLflow answers 401 through the portal | `proxy.secret` changed after `setup-ood.sh` ran | rerun `3-mlflow/2-deploy.sh`, then `setup-ood.sh` |
| `/auth` gives 404 | Apache has no `/auth` proxy yet | rerun `setup-ood.sh` |
| `/auth` gives 503 | Keycloak down, or Apache proxies to an old address | check Keycloak; rerun `setup-ood.sh` |
| Keycloak: `Invalid parameter: redirect_uri` | the browser uses a host other than `OOD_SERVERNAME` | set `OOD_SERVERNAME` to what users type; rerun |
