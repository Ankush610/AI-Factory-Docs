# Keycloak

Issues in [`6-keycloak`](https://github.com/Ankush610/OOD-Install/tree/main/6-keycloak). How to read this page:
[Known issues](index.md#how-to-read-these-pages).

| Issue | Type | Confidence |
|---|---|---|
| [Fresh-install order isn't stated](#fresh-install-order-isnt-stated) | Docs | Confirmed |
| [Backup command misses the port](#backup-command-misses-the-port) | Docs | Inferred |
| [Plain HTTP between the login node and master](#plain-http-between-the-login-node-and-master) | Security | Confirmed |
| [Admin console reachable by anyone who reaches the portal](#admin-console-reachable-by-anyone-who-reaches-the-portal) | Security | Inferred |
| [Image isn't rebuilt when only the path or Containerfile changes](#image-isnt-rebuilt-when-only-the-path-or-containerfile-changes) | Bug | Inferred |
| [README says MLflow and Model Hub use Keycloak directly](#readme-says-mlflow-and-model-hub-use-keycloak-directly) | Docs | Confirmed |
| [Missing test script](#missing-test-script) | Docs | Confirmed |
| [Smaller items](#smaller-items) | Mixed | — |

## Fresh-install order isn't stated

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `site.conf:42`, `2-ood/setup-ood.sh:28-32`, `6-keycloak/1-install.sh:30-32`
- **What happens:** the portal must be installed with `OOD_AUTH=ldap` before Keycloak, then switched. The folder
  README assumes this but never says it, and `site.conf` ships `keycloak`. See [Portal](portal.md#first-install-cant-use-the-shipped-ood_authkeycloak).

## Backup command misses the port

- **Type:** Docs · **Confidence:** Inferred (not run) · **Where:** `6-keycloak/README.md:94`, `6-keycloak/1-install.sh:105`
- **What happens:** the README's `podman exec keycloak-db pg_dump -U keycloak keycloak` has no `-p`, but Postgres
  listens on 5433. `pg_dump` tries 5432 and likely fails. No restore procedure is documented.
- **Workaround:** add `-p 5433` (see [Install → Keycloak](../admin/install/keycloak.md#back-up)).

## Plain HTTP between the login node and master

- **Type:** Security · **Confidence:** Confirmed · **Where:** `6-keycloak/1-install.sh:140-141` (`# ponytail:`)
- **What happens:** Apache on the login node talks to Keycloak over plain HTTP, so the login form (passwords) and
  tokens cross the cluster network unencrypted when the login node is a separate machine.
- **Workaround:** acceptable on a private rack. Otherwise give Keycloak a certificate (`KC_HTTPS_*`) and point
  `KEYCLOAK_URL` at https.

## Admin console reachable by anyone who reaches the portal

- **Type:** Security · **Confidence:** Inferred · **Where:** `2-ood/setup-ood.sh:133-137`
- **What happens:** `https://<portal>/auth/admin` has no network restriction. It still needs Keycloak's admin
  password, and brute-force protection is on with Keycloak's defaults.
- **Workaround:** restrict `/auth/admin` in Apache to the admins' network.

## Image isn't rebuilt when only the path or Containerfile changes

- **Type:** Bug · **Confidence:** Inferred · **Where:** `6-keycloak/1-install.sh:71-72`, `6-keycloak/image/Containerfile:6-7`
- **What happens:** the image is built only when its tag is missing from the registry. Changing `KEYCLOAK_PATH`
  (built into the image) or editing the Containerfile without a version change keeps the old image. The `-rudra`
  tag suffix isn't explained anywhere.
- **Workaround:** delete the tag from the registry, or change the tag, then rerun `1-install.sh`.

## README says MLflow and Model Hub use Keycloak directly

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `6-keycloak/README.md:3`, `site.conf:50`
- **What happens:** the text says the portal, MLflow and Model Hub "send the browser to Keycloak". Only the client
  `ood` exists. MLflow and Model Hub ride on the portal's session through `/node`. Is a client per app planned?

## Missing test script

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `6-keycloak/README.md:117`
- **What happens:** the troubleshooting table points to `testing/keycloak-passwords/run.sh`, which isn't in the repo.

## Smaller items

- **Password file list** (Docs, Confirmed): the README lists `admin`, `db` and `bootstrap.pass` but not
  `ldap.pass`. `bootstrap.pass` and the bootstrap settings stay after the bootstrap admin is deleted.
- **Port check can be skipped** (Bug, Inferred): `systemctl is-active` on both units succeeds if **either** is
  active, so the free-port check is skipped in a half-installed state (`1-install.sh:34`).
- **Keycloak's own timeouts** (Gap, Confirmed): session and brute-force limits are Keycloak defaults. The "8 h" in
  the docs is the portal's own session setting.
- **Data folder ownership** (Security, Inferred): `KEYCLOAK_DATA` is owned by 949 while the root-only `*.pass` files
  sit inside; the container user can't read them but could rename or delete them.
- **Postgres on `/home/apps`** (Gap, Inferred): it is local disk only while master is the `/home` NFS server. Nothing
  guards against Postgres ending up on NFS.
- **LDAP password in the database** (Security, Inferred): the LDAP account's password is stored in Keycloak's
  database, so every `pg_dump` contains it.
- **Design link** (Docs, Confirmed): the README links to `OOD-Install/project/`, which is private: most readers can't open it.
