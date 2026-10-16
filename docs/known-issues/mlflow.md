# MLflow

Issues in [`3-mlflow`](https://github.com/Ankush610/OOD-Install/tree/main/3-mlflow). How to read this page:
[Known issues](index.md#how-to-read-these-pages).

| Issue | Type | Confidence |
|---|---|---|
| [Keycloak with an empty `MLFLOW_PREFIX` breaks Open MLflow](#keycloak-with-an-empty-mlflow_prefix-breaks-open-mlflow) | Bug | Confirmed |
| [Secrets briefly on a command line](#secrets-briefly-on-a-command-line) | Security | Confirmed |
| [`WORKFLOW.md` is out of date for Keycloak](#workflowmd-is-out-of-date-for-keycloak) | Docs | Confirmed |
| [Direct-mode link points at the browser's host](#direct-mode-link-points-at-the-browsers-host) | Bug | Inferred |
| ["Job login: OK" only checks the file exists](#job-login-ok-only-checks-the-file-exists) | Bug | Confirmed |
| [Page unstyled until Model Hub is installed](#page-unstyled-until-model-hub-is-installed) | Gap | Inferred |
| [No clean way to roll out a rebuilt image](#no-clean-way-to-roll-out-a-rebuilt-image) | Gap | Inferred |
| [Backup steps are incomplete](#backup-steps-are-incomplete) | Docs | Confirmed |
| [Admin password can drift from `admin.pass`](#admin-password-can-drift-from-adminpass) | Bug | Inferred |
| [Smaller items](#smaller-items) | Mixed | — |

## Keycloak with an empty `MLFLOW_PREFIX` breaks Open MLflow

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `3-mlflow/4-install-ood-app.sh:21-22`,
  `2-ood/setup-ood.sh:109-110`
- **What happens:** the page turns on single sign-on from `OOD_AUTH` alone. With `OOD_AUTH=keycloak` and
  `MLFLOW_PREFIX=""` (which the README suggests for new clusters), **Open MLflow** opens `location.origin + "/"`:
  the portal dashboard. `setup-ood.sh` skips single sign-on in that case, so MLflow is unreachable from the page.
- **Workaround:** keep `MLFLOW_PREFIX=/node/$MASTER_IP/$MLFLOW_PORT` when using Keycloak.

## Secrets briefly on a command line

- **Type:** Security · **Confidence:** Confirmed · **Where:** `3-mlflow/3-sync-tokens.sh:19`, `3-mlflow/2-deploy.sh:66`
- **What happens:** new job tokens are passed as `python3 -c … "$3" "$4"` arguments, and the Flask key with
  `kubectl create secret --from-literal`. Both are visible in `ps` on master while they run. This breaks the repo's
  own rule ("secrets never go on a command line"). On the test cluster master is also the login node, so users can
  run `ps` there.

## `WORKFLOW.md` is out of date for Keycloak

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `3-mlflow/WORKFLOW.md:14,27,36,74,78,133`; `README.md:102`
- **What happens:** it says Open MLflow always goes straight to port 30500 and that single sign-on is a future fix.
  With `OOD_AUTH=keycloak` the button opens MLflow through the portal. The top-level README's `-L 30500` tunnel note
  may also be out of date. It also says the tracking URI "never changes", but it changes with `MLFLOW_PREFIX`,
  `MASTER_IP` or `MLFLOW_PORT`.

## Direct-mode link points at the browser's host

- **Type:** Bug · **Confidence:** Inferred · **Where:** `3-mlflow/ood-app/mlflow_k8s/passenger_wsgi.py:112-115`
- **What happens:** without single sign-on, **Open MLflow** goes to `http://<browser's hostname>:30500…`, while the
  page's note names `MASTER_IP`. With a separate login node, the link points at the login node, where MLflow doesn't
  run.

## "Job login: OK" only checks the file exists

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `3-mlflow/ood-app/mlflow_k8s/passenger_wsgi.py:102`
- **What happens:** the page shows OK when `~/.mlflow/credentials` exists, even if the token is stale and jobs get 401.

## Page unstyled until Model Hub is installed

- **Type:** Gap · **Confidence:** Inferred · **Where:** `3-mlflow/ood-app/mlflow_k8s/passenger_wsgi.py:17,119-121,164`
- **What happens:** the page loads Bootstrap and its stylesheet from `/pun/sys/model_hub/`, but MLflow is installed
  before Model Hub. Until then the page is unstyled and the FAQ accordion doesn't open.

## No clean way to roll out a rebuilt image

- **Type:** Gap · **Confidence:** Inferred · **Where:** `site.conf:71`, `3-mlflow/mlflow.yaml:54`, `3-mlflow/1-build-image.sh:5`
- **What happens:** the script says to bump the tag after an image change, but the tag is built from
  `MLFLOW_VERSION` + `-sso`, so there is no separate knob. With `imagePullPolicy: IfNotPresent`, a rebuilt image
  under the same tag is not pulled.
- **Related:** the base image `python:3.12-slim` is not pinned; `--from-venv` may write an `ldap3` pin that clashes
  with the Containerfile's `ldap3==2.9.1`.

## Backup steps are incomplete

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `3-mlflow/README.md:56`
- **What happens:** "back up `MLFLOW_DATA` as one piece" doesn't say whether to stop the pod or how to get a
  consistent SQLite copy, and leaves out `admin.pass` and `proxy.secret`.

## Admin password can drift from `admin.pass`

- **Type:** Bug · **Confidence:** Inferred · **Where:** `3-mlflow/2-deploy.sh:39-49`
- **What happens:** if someone changes the admin password in the MLflow UI, `admin.pass` goes stale. Then
  `3-sync-tokens.sh`, `remove-user.sh` and the public-models job (which log in as admin) fail.
- **Workaround:** don't change the admin password in the UI; if you do, update `admin.pass` to match.

## Smaller items

- **No host check** in `1-build-image.sh` and `3-sync-tokens.sh`; the latter only works on master
  (`ldap://localhost`). (Confirmed)
- **`:636` is appended to `LDAP_URI`** in `mlflow.yaml:84`; a `LDAP_URI` that already has a port would break. (Inferred)
- **Install order with Keycloak:** the `setup-ood.sh` rerun after `2-deploy.sh` is only hinted at by a message.
  (Confirmed; covered in [Install → MLflow](../admin/install/mlflow.md#2-deploy-master).)
- **Generated Apache config** holds the proxy secret; its mode is not set by the repo. See
  [Portal](portal.md#secret-files-without-an-explicit-mode). (Inferred)
