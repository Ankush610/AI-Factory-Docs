# Known issues

Bugs, errors, security gaps and documentation mismatches found by reading the
[OOD-Install](https://github.com/Ankush610/OOD-Install) code, one page per component.

These pages are a to-do list. When an issue is fixed in OOD-Install, delete it here in the same change.

## How to read these pages

Every issue has the same fields:

| Field | Values |
|---|---|
| **Type** | **Bug**: the code does the wrong thing. **Security**: a weakness someone could use. **Docs**: a README, comment or message says something the code doesn't do. **Gap**: something missing (a procedure, a check, a setting). **Question**: needs a decision from the maintainers |
| **Confidence** | **Confirmed**: read directly in the code. **Inferred**: follows from the code, but not tested on a running cluster. Test it before you rely on it |
| **Where** | `file:line` in OOD-Install (at the commit reviewed in October 2026) |
| **What happens** | the effect, in plain words |
| **Workaround / Fix** | what to do until the code is fixed, when there is something |

## Fix these first

The issues with the biggest effect on a new site or on safety.

| # | Issue | Type | Page |
|---|---|---|---|
| 1 | A fresh install can't use the shipped `OOD_AUTH=keycloak`; nothing says to start with `ldap` | Bug | [Portal](portal.md#first-install-cant-use-the-shipped-ood_authkeycloak) |
| 2 | No network lock: any pod can call a model directly and skip the gateway | Security | [Model Hub](model-hub.md#no-network-lock-pods-can-skip-the-gateway) |
| 3 | The registry has no login, allows deletes, and no firewall rule protects it | Security | [Registry](registry.md#no-login-deletes-on-no-firewall-rule) |
| 4 | LDAP (389 and 636) is open to every source, and anyone can list every user | Security | [LDAP](ldap.md#ldap-open-to-every-source) |
| 5 | The top-level README's password reset uses plain LDAP and fails | Docs | [LDAP](ldap.md#top-level-readme-resets-passwords-over-plain-ldap) |
| 6 | Certificate renewal (portal, LDAP) isn't documented; rerunning the scripts renews nothing | Docs | [Install and docs](install-and-docs.md#certificate-renewal-isnt-really-documented) |
| 7 | No backup or restore procedure for LDAP, MLflow, Keycloak, Prometheus or Model Hub state | Gap | [Install and docs](install-and-docs.md#no-backup-or-restore-procedure) |
| 8 | Users' Kubernetes certificates expire after a year and nothing renews them | Gap | [Model Hub](model-hub.md#certificates-arent-renewed-automatically) |
| 9 | The Jupyter page can break for a user after a leftover kernels folder | Bug | [Jupyter](jupyter.md#the-page-can-break-on-a-leftover-kernels-folder) |
| 10 | With Keycloak and an empty `MLFLOW_PREFIX`, **Open MLflow** goes to the dashboard | Bug | [MLflow](mlflow.md#keycloak-with-an-empty-mlflow_prefix-breaks-open-mlflow) |
| 11 | No email or paging for alerts, and nothing watches the watcher | Gap | [Monitoring](monitoring.md#no-notifications-and-no-watch-on-the-watcher) |
| 12 | The GPU type users pick in Model Hub is ignored; several GPU types break GPU endpoints | Bug | [Model Hub](model-hub.md#the-gpu-type-a-user-picks-is-ignored) |

## By component

| Page | What it covers |
|---|---|
| [Install and docs](install-and-docs.md) | install order, prerequisites, host checks, stale READMEs, backups, adding nodes |
| [Security](security.md) | every security finding from all components, in one table |
| [Registry](registry.md) | `0-registry`: the image registry and `k8s-image` |
| [LDAP](ldap.md) | `1-ldap`: users, passwords, teams, removal |
| [Portal](portal.md) | `2-ood`: Open OnDemand, login modes, certificate, branding |
| [MLflow](mlflow.md) | `3-mlflow`: server, login, tokens, portal page |
| [VS Code](vscode.md) | `4-vscode`: code-server, viewer partition, session page |
| [Model Hub](model-hub.md) | `5-model-hub`: tenancy, images, endpoints, gateway, public models |
| [Keycloak](keycloak.md) | `6-keycloak`: single sign-on |
| [Jupyter](jupyter.md) | `7-jupyter`: JupyterLab sessions and kernels |
| [Admin page](admin-page.md) | `8-admin`: the root helper and the Admin page |
| [Monitoring](monitoring.md) | `9-monitoring`: Prometheus, collectors, alerts, Cluster monitor |
