# Portal (Open OnDemand)

Issues in [`2-ood`](https://github.com/Ankush610/OOD-Install/tree/main/2-ood). How to read this page:
[Known issues](index.md#how-to-read-these-pages).

| Issue | Type | Confidence |
|---|---|---|
| [First install can't use the shipped `OOD_AUTH=keycloak`](#first-install-cant-use-the-shipped-ood_authkeycloak) | Bug | Confirmed |
| [Certificate renewal isn't documented](#certificate-renewal-isnt-documented) | Docs | Confirmed |
| [Changing `OOD_VERSION` doesn't upgrade](#changing-ood_version-doesnt-upgrade) | Gap | Confirmed |
| [Folder README describes `ldap` mode only](#folder-readme-describes-ldap-mode-only) | Docs | Confirmed |
| [Keycloak admin console reachable through `/auth`](#keycloak-admin-console-reachable-through-auth) | Security | Inferred |
| [Secret files without an explicit mode](#secret-files-without-an-explicit-mode) | Security | Inferred |
| [No way in when Keycloak is down, except switching mode](#no-way-in-when-keycloak-is-down-except-switching-mode) | Docs | Confirmed |
| [Login node also needs internet](#login-node-also-needs-internet) | Docs | Confirmed |
| [Card texts promise things the install doesn't set up](#card-texts-promise-things-the-install-doesnt-set-up) | Docs | Confirmed |
| [Open questions](#open-questions) | Question | — |

## First install can't use the shipped `OOD_AUTH=keycloak`

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `site.conf:42`, `2-ood/setup-ood.sh:28-32`,
  `6-keycloak/1-install.sh:31-32`
- **What happens:** `site.conf` ships `OOD_AUTH=keycloak`. In that mode `setup-ood.sh` stops until Keycloak's realm
  and client secret exist. But `6-keycloak/1-install.sh` needs the portal's certificate, which `setup-ood.sh` makes.
  The top-level README never says to start with `ldap`. Also, the switch command in `6-keycloak/README.md:33`
  (`sed 's/^OOD_AUTH=ldap /…/'`) needs a trailing space after `ldap`, so it may not match.
- **Workaround:** the order in the [Admin guide](../admin/index.md#install-order): `ldap` first, then Keycloak, then switch.
- **Related:** the README says the script "asks once for the admin's web password", but in `keycloak` mode it doesn't.

## Certificate renewal isn't documented

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `2-ood/setup-ood.sh:53`; `9-monitoring/README.md:122`
- **What happens:** the monitoring runbook says to renew the portal certificate by rerunning `setup-ood.sh`. The
  script never replaces an existing certificate, so nothing changes. The certificate also has no subjectAltName
  (CN only), which some clients reject (inferred).
- **Workaround (inferred from the code):** on the login node delete `OOD_CERT` and `/etc/pki/tls/private/ood.key`,
  rerun `setup-ood.sh`, then rerun `6-keycloak/1-install.sh` on master so Keycloak trusts the new certificate.

## Changing `OOD_VERSION` doesn't upgrade

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `2-ood/setup-ood.sh:41-42`
- **What happens:** the release RPM is installed only if none is installed, so a new `OOD_VERSION` doesn't switch
  repositories on an existing install. There is no upgrade procedure.

## Folder README describes `ldap` mode only

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `2-ood/README.md`
- **What happens:**
    - "What it does" lists 5 steps; the script has 10.
    - The package list leaves out `httpd-tools` and `mod_auth_openidc`.
    - "Success looks like" gives only the `ldap` result (401) and says **Jobs → Active Jobs** loads, but the Jobs
      menu was removed.
    - The variable table misses about 10 variables the script uses (`OOD_AUTH`, `OOD_CERT`, `KEYCLOAK_*`, …).
- **Workaround:** use [Install → Portal](../admin/install/portal.md), which covers both modes.

## Keycloak admin console reachable through `/auth`

- **Type:** Security · **Confidence:** Inferred · **Where:** `2-ood/setup-ood.sh:130-138`
- **What happens:** the `/auth` proxy is always written, also in `ldap` mode, and is not behind the portal's
  login (Keycloak is the login page). So `https://<portal>/auth/admin` is reachable by anyone who can reach port 443.
  It still asks for Keycloak's admin password.
- **Workaround:** block `/auth/admin` in Apache for addresses outside the admin network.

## Secret files without an explicit mode

- **Type:** Security · **Confidence:** Inferred (not checked on a live system) · **Where:** `2-ood/setup-ood.sh:59,141`
- **What happens:**
    - `ood_portal.yml` is mode 600 because it holds the OIDC and MLflow secrets. The Apache config that
      `update_ood_portal` generates from it holds the same secrets, and the script doesn't check its mode.
    - `/etc/ood/htpasswd` gets no explicit mode, and an old one is reused silently when you switch back to `ldap`.
- **Workaround:** check with `ls -l /etc/httpd/conf.d/ood-portal.conf /etc/ood/htpasswd` and `chmod 600` if needed.

## No way in when Keycloak is down, except switching mode

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `site.conf:41`, `6-keycloak/README.md:112`
- **What happens:** `OOD_ADMIN` "works even when LDAP is down", but in `keycloak` mode it is a Keycloak user. If
  Keycloak is down, the only way back in is setting `OOD_AUTH=ldap` and rerunning `setup-ood.sh`.

## Login node also needs internet

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `2-ood/setup-ood.sh:37-42`; `README.md:33`
- **What happens:** the prerequisites list internet on master only, but `setup-ood.sh` downloads repositories and
  RPMs on the login node.

## Card texts promise things the install doesn't set up

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `2-ood/branding/_ai_tools.html.erb:14-15,28-29`
- **What happens:** the Files card mentions "scratch" storage and the Jupyter card "datasets already in place".
  Nothing in the repo creates scratch or dataset folders.

## Open questions

- Card visibility relies on Open OnDemand listing only apps the user can read (`SysRouter.apps`). This is Open
  OnDemand behaviour, not shown in the repo; confirm it for 4.2.
- `nginx_stage app --user=$OOD_ADMIN` may fail if `OOD_ADMIN` has never logged in to the portal.
- `KEEP_LOCAL` and `OOD_ADMIN` are both `admin`. Must `OOD_ADMIN` always be in `KEEP_LOCAL`?
- The footer year ("© 2026") is hard-coded in `ai-factory.css`.
