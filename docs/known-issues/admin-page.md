# Admin page

Issues in [`8-admin`](https://github.com/Ankush610/OOD-Install/tree/main/8-admin). How to read this page:
[Known issues](index.md#how-to-read-these-pages).

| Issue | Type | Confidence |
|---|---|---|
| [The helper runs scripts its owner can edit](#the-helper-runs-scripts-its-owner-can-edit) | Security | Confirmed |
| [Visibility depends on the admin's primary group](#visibility-depends-on-the-admins-primary-group) | Security | Inferred |
| [Teams tab fails without Kubernetes](#teams-tab-fails-without-kubernetes) | Bug | Confirmed |
| [Some slurm-bridge jobs can't be stopped from the page](#some-slurm-bridge-jobs-cant-be-stopped-from-the-page) | Bug | Inferred |
| [Timeouts show as "bad request"](#timeouts-show-as-bad-request) | Bug | Inferred |
| [Audit log gaps](#audit-log-gaps) | Gap | Confirmed |
| [Smaller items](#smaller-items) | Mixed | — |

## The helper runs scripts its owner can edit

- **Type:** Security · **Confidence:** Confirmed · **Where:** `8-admin/1-install.sh:25-26` (`# ponytail:`)
- **What happens:** the root helper runs scripts from the OOD-Install folder, which its owner can edit. That is fine
  while the only admin already has full sudo. Before anyone else gets admin rights, the folder must be root-owned.
  The code calls this "step 9", which isn't defined in the repo.

## Visibility depends on the admin's primary group

- **Type:** Security · **Confidence:** Inferred · **Where:** `8-admin/2-install-ood-app.sh:20,31-33,44-47`
- **What happens:** the app folder is readable by `OOD_ADMIN`'s primary group. If that group is shared (for
  example `users`), other members see the app in the portal. Its backend still answers 403. The install check tests
  only the first LDAP user.
- **Workaround:** give `OOD_ADMIN` a private primary group.

## Teams tab fails without Kubernetes

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `app.js:225`, `aistack-admin.py:133-134`
- **What happens:** the Teams tab also loads `users`, which always calls `kubectl`. Without Model Hub or with
  Kubernetes down, the Teams tab fails even though teams are pure LDAP.

## Some slurm-bridge jobs can't be stopped from the page

- **Type:** Bug · **Confidence:** Inferred · **Where:** `aistack-admin.py:208-210,238-239`, `app.js:146-151`
- **What happens:** a job in the `slurm-bridge` partition that isn't matched to an endpoint (Kubernetes down, or the
  annotation missing) shows the Slurm user and job name. Its **Stop** calls `stop-endpoint` with those, which fails;
  `cancel-job` refuses bridge jobs.
- **Workaround:** `scancel <job>` on master.

## Timeouts show as "bad request"

- **Type:** Bug · **Confidence:** Inferred · **Where:** `passenger_wsgi.py:31-32,124-125`
- **What happens:** a helper call over 900 s shows as `400 bad request: …`. The change may still finish on master.
- **Workaround:** check the Log tab before retrying.

## Audit log gaps

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `aistack-admin.py:285-290,311-316`
- **What happens:**
    - Reads (who looked at the user list) and failed reads are not logged.
    - The log has no rotation and grows forever.
    - Each install run logs a refused `id` attempt by `OOD_ADMIN` (the install's own check), which can look like an
      intrusion in the Log tab.

## Smaller items

- `site.conf:40` says `ADMIN_NOTICES` holds "one file per job"; endpoint stops write one too. (Docs)
- A job named `vscode`, `jupyter` or `endpoint` by its user is labelled as that kind under the bell. (Bug, minor)
- The backend's `api/endpoints` route is never used by the page. (Docs)
- The dialog says "Adding or removing someone can take a minute" for every action, including stops. (Cosmetic)
- Notices of removed users stay as small files (`# ponytail:` in `aistack-admin.py:251`). (Gap)
- The admin key has no passphrase (it is used unattended); anyone who becomes `OOD_ADMIN` or root on the login node
  can run every action. (By design)
