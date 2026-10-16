# LDAP

Issues in [`1-ldap`](https://github.com/Ankush610/OOD-Install/tree/main/1-ldap). How to read this page:
[Known issues](index.md#how-to-read-these-pages).

| Issue | Type | Confidence |
|---|---|---|
| [Top-level README resets passwords over plain LDAP](#top-level-readme-resets-passwords-over-plain-ldap) | Docs | Confirmed |
| [Rerunning `1-server.sh` doesn't renew the certificate](#rerunning-1-serversh-doesnt-renew-the-certificate) | Docs | Confirmed |
| [LDAP open to every source](#ldap-open-to-every-source) | Security | Confirmed |
| [A failed removal can't be finished by a rerun](#a-failed-removal-cant-be-finished-by-a-rerun) | Bug | Confirmed |
| [Removal doesn't stop processes on master](#removal-doesnt-stop-processes-on-master) | Bug | Confirmed |
| [No backup or restore procedure](#no-backup-or-restore-procedure) | Gap | Confirmed |
| [Lost Directory Manager password file](#lost-directory-manager-password-file) | Gap | Confirmed |
| [User ID clash check covers master only](#user-id-clash-check-covers-master-only) | Bug | Confirmed |
| [No password policy](#no-password-policy) | Security | Confirmed |
| [Script hints skip the login node](#script-hints-skip-the-login-node) | Docs | Confirmed |
| [Small README and code differences](#small-readme-and-code-differences) | Docs | Confirmed |

## Top-level README resets passwords over plain LDAP

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `README.md:83`
- **What happens:** the "reset a password" row uses `ldappasswd -H ldap://localhost`. 389 DS refuses password
  changes on a plain connection with `Confidentiality required (13)`. Every script uses ldaps.
- **Workaround:** use `set-password.sh`, or the ldaps command in
  [Install → LDAP](../admin/install/ldap.md#passwords).

## Rerunning `1-server.sh` doesn't renew the certificate

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `1-ldap/1-server.sh:27-28`; `9-monitoring/README.md:122`
- **What happens:** the monitoring runbook says to renew the LDAP certificate by rerunning `1-server.sh`. That script
  skips an existing instance and only copies the CA again, so nothing is renewed. The certificate's lifetime is not
  set in the code either.
- **Workaround:** none documented. Renew the certificate with 389 DS's own tools (`dsctl`/`certutil`) and copy the
  new CA to `LDAP_CA`.

## LDAP open to every source

- **Type:** Security · **Confidence:** Confirmed · **Where:** `1-ldap/1-server.sh:73,88`
- **What happens:** ports 389 (plain) and 636 are opened to every source, and the read rule allows anyone to read
  every attribute except passwords. Anyone on the network can list all users and groups.
- **Workaround:** restrict 389 to master itself and 636 to the cluster nodes with firewalld rich rules.

## A failed removal can't be finished by a rerun

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `1-ldap/remove-user.sh:32`; `1-ldap/README.md:93`
- **What happens:** if `remove-user.sh` fails **after** the LDAP delete, a rerun stops at `No LDAP user`. The
  Keycloak removal, the second "stop everything" and the home deletion are not redone. The README says only
  steps 7–8 were left, which leaves out the Keycloak removal and step 9.
- **Workaround:** finish by hand: `6-keycloak/kc.py remove-user <name>`, `scancel -u <uid>`, `pkill -U <uid>` on each
  node, `sss_cache -E`, and delete the home if wanted.

## Removal doesn't stop processes on master

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `1-ldap/remove-user.sh:56`
- **What happens:** "stop everything" covers `LOGIN_NODES` and `COMPUTE_NODES`. When the login node is a separate
  machine, the person's processes on master are not killed.

## No backup or restore procedure

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `site.conf:27`
- **What happens:** `site.conf` says to back up `LDAP_LAST_UID_FILE` with LDAP, but the repo has no command to
  export or restore the 389 DS directory.
- **Workaround:** export with `dsconf <instance> backend export` or `db2ldif`, and keep
  `/root/.ldap-last-uid` and `/root/.ldap-dm.pass` with it.

## Lost Directory Manager password file

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `1-ldap/1-server.sh:19-21,55-56`
- **What happens:** if `LDAP_DM_PASS_FILE` is lost, `1-server.sh` writes a new random password that doesn't match
  the instance, and stops. Recovery isn't documented.

## User ID clash check covers master only

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `1-ldap/add-user.sh:27-30`
- **What happens:** when `add-user.sh` picks the next ID, it checks master's local accounts and LDAP, not the
  compute nodes. A local account on a compute node with that ID is only caught when `2-client.sh` runs there again.

## No password policy

- **Type:** Security · **Confidence:** Confirmed · **Where:** `1-ldap/add-user.sh:17`, `1-ldap/set-password.sh:17`
- **What happens:** the 8-character minimum applies only to passwords read from standard input. The interactive
  `ldappasswd -S` path has no check, and no 389 DS password policy (length, expiry, lockout) is set up.

## Script hints skip the login node

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `1-ldap/1-server.sh:106`, `1-ldap/2-client.sh:4`
- **What happens:** both scripts tell you to run `2-client.sh` "on master, then the compute nodes". The READMEs
  correctly include the login node too.

## Small README and code differences

- **Type:** Docs · **Confidence:** Confirmed
- The README says Check covers master and the compute nodes; the code also checks `LOGIN_NODES` (`1-ldap/menu.sh:155`).
- `menu.sh:22` defines an `L` array that is never used.
- The README's "Run by hand" block leaves out `add-user.sh --password-stdin` and `remove-user.sh --yes`.
- `add-user.sh`, `set-password.sh`, `group.sh` and `import-local-users.sh` have no host check, although they only
  work on master (see [Install and docs](install-and-docs.md#not-every-script-checks-its-host)).
