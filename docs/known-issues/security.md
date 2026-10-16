# Security

Every security finding from the component pages, in one place. The platform's security design is explained in
[Security and isolation](../how-it-works/11-security-and-isolation.md); this page lists where the code falls short
of it. How to read this page: [Known issues](index.md#how-to-read-these-pages).

Most findings matter only when someone can reach the cluster network, or when the login node is a separate machine.
On a private, single-rack network several are accepted risks; they are listed so each site can decide.

## Network exposure

| Finding | Risk | Confidence | Details |
|---|---|---|---|
| No network lock between pods | any pod can call a model's internal address directly, skipping keys and sharing rules | Confirmed | [Model Hub](model-hub.md#no-network-lock-pods-can-skip-the-gateway) |
| Registry: no login, deletes on, no firewall rule | anyone on the network can push, overwrite or delete images, including MLflow's and Keycloak's | Confirmed | [Registry](registry.md#no-login-deletes-on-no-firewall-rule) |
| LDAP 389/636 open to every source | anyone on the network can list every user and group | Confirmed | [LDAP](ldap.md#ldap-open-to-every-source) |
| Keycloak admin console reachable through the portal | `https://<portal>/auth/admin` is reachable by anyone who reaches port 443 (still password-protected) | Inferred | [Keycloak](keycloak.md#admin-console-reachable-by-anyone-who-reaches-the-portal) |
| VS Code and Jupyter listen on `0.0.0.0` | the per-session password or token is the only barrier, for anyone on the network and any portal user (`/node`, `/rnode`) | Inferred | [VS Code](vscode.md#sessions-are-protected-only-by-their-password), [Jupyter](jupyter.md#sessions-are-protected-only-by-their-token) |
| No firewall rule for the gateway | decide which networks may call models | Confirmed | [Model Hub](model-hub.md#no-firewall-rule-for-the-gateway) |

## Traffic in clear text

| Finding | Risk | Confidence | Details |
|---|---|---|---|
| Login node → Keycloak over plain HTTP | with a separate login node, the login form (passwords) and tokens cross the rack network unencrypted | Confirmed | [Keycloak](keycloak.md#plain-http-between-the-login-node-and-master) |
| MLflow over plain HTTP | job tokens, and in `ldap` mode users' passwords, cross the network unencrypted | Confirmed | [MLflow](mlflow.md) |
| Jupyter token in the URL | the token can land in Apache's log and browser history; valid up to 12 h | Inferred | [Jupyter](jupyter.md#session-token-travels-in-the-url) |

## Secrets and privileges

| Finding | Risk | Confidence | Details |
|---|---|---|---|
| The gateway may read every user's Secrets | includes each user's MLflow token | Inferred | [Model Hub](model-hub.md#the-gateway-can-read-every-users-mlflow-token) |
| MLflow secrets briefly on a command line | job tokens and the Flask key are visible in `ps` on master while the scripts run | Confirmed | [MLflow](mlflow.md#secrets-briefly-on-a-command-line) |
| Admin helper runs scripts its owner can edit | whoever can edit the OOD-Install folder can make the root helper run anything | Confirmed | [Admin page](admin-page.md#the-helper-runs-scripts-its-owner-can-edit) |
| Admin-only apps rely on `OOD_ADMIN`'s primary group | a shared group shows the Admin page and Cluster monitor to others (their backends still check) | Inferred | [Admin page](admin-page.md#visibility-depends-on-the-admins-primary-group) |
| Portal secret files without an explicit mode | the generated Apache config holds the OIDC and MLflow secrets; its mode isn't set by the repo | Inferred | [Portal](portal.md#secret-files-without-an-explicit-mode) |
| No LDAP password policy | the interactive path has no length check; no expiry or lockout in 389 DS | Confirmed | [LDAP](ldap.md#no-password-policy) |
| Keycloak dump contains the LDAP account's password | every `pg_dump` must be stored like a secret | Inferred | [Keycloak](keycloak.md#smaller-items) |

## Supply chain

| Finding | Risk | Confidence | Details |
|---|---|---|---|
| code-server downloaded without a checksum | a tampered download would be installed for every user | Confirmed | [VS Code](vscode.md#no-checksum-for-the-code-server-download) |
| Floating base images (`registry:2`, `python:3.12-slim` for MLflow) | rebuilds may silently pick up different software | Confirmed | [Registry](registry.md#registry-image-not-pinned), [MLflow](mlflow.md#no-clean-way-to-roll-out-a-rebuilt-image) |

## By design (not bugs)

- **Prometheus has no password.** Portal apps run as the user, so a secret they read is readable by every user. The
  firewall and a read-only API protect it instead.
- **The Admin page's SSH key has no passphrase.** It is used unattended; it can only run the admin helper, only from
  the login node.
- **Self-signed certificates** for the portal and gateway until the customer provides real ones.
