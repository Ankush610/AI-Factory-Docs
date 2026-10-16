# Scripts

Every install and admin script in OOD-Install: where it runs, whether it needs root, and what it does. Run them
from the OOD-Install folder on the shared `/home`.

**Conventions shared by every script:**

- Stops at the first error (`set -euo pipefail`), prints numbered `== N.` steps, and ends with a check that says
  what to expect. Safe to rerun.
- Reads every site value from `site.conf`; nothing is hardcoded.
- Prints `Run with sudo.` or `Run this on <host>.` when started the wrong way (some don't check the host yet; see
  [known issues](../known-issues/install-and-docs.md#not-every-script-checks-its-host)).
- Never puts a secret on a command line: secrets go through files or standard input.
- Master reaches other nodes as root over SSH, with a 5-second timeout; unreachable nodes are skipped and listed.
- After a portal app or config change: **Restart Web Server** in the portal.

| Script | Runs on | Root | What it does |
|---|---|---|---|
| **0-registry** | | | |
| `setup-registry.sh` | master | yes | registry container, containerd trust on Kubernetes nodes, podman trust on login nodes, `k8s-image` |
| **1-ldap** | | | |
| `1-server.sh` | master | yes | 389 DS, `ou=People`/`ou=Groups`, access rules, CA to `LDAP_CA` |
| `import-local-users.sh [--dry-run]` | master | yes | optional: local people into LDAP with the same ID and password hash |
| `2-client.sh` | **every** node | yes | ID clash check, SSSD, authselect |
| `add-user.sh <name> [--password-stdin]` | master | yes | a new person and everything they need |
| `remove-user.sh <name> [--delete-home] [--delete-mlflow] [--yes]` | master | yes | someone leaves |
| `set-password.sh <name>` | master | yes | one-time password from standard input |
| `group.sh list\|create\|add\|remove\|delete` | master | yes | teams for Model Hub sharing |
| `menu.sh` | master (terminal) | yes | text menu over all of the above, plus password test and local-copy check |
| **2-ood** | | | |
| `setup-ood.sh` | login node | yes | Open OnDemand, login mode, certificate, cluster file, branding, bell |
| **3-mlflow** | | | |
| `1-build-image.sh [--from-venv <dir>]` | master | **no** | build and push the MLflow image |
| `2-deploy.sh` | master | yes | `mlflow` account, data folder, deployment; copies the proxy secret to the login node |
| `3-sync-tokens.sh` | master | yes | a job token for every LDAP user |
| `4-install-ood-app.sh` | login node | yes | the MLflow portal page |
| **4-vscode** | | | |
| `1-install-code-server.sh` | master | yes | code-server on the shared `/home` |
| `2-slurm-viewer.sh` | master | yes | the `viewer` partition on the login node; `slurm.conf` to every node |
| `3-install-ood-app.sh` | login node | yes | the VS Code portal page |
| **5-model-hub** | | | |
| `1-setup.sh` | master | yes | `modelhub` account, pod policies, GPU classes, slurm-bridge scheduler fix, `model-register`, public-models timer |
| `2-sync-users.sh [<user>] [--gpus N\|--max-hours H\|--reset\|--show]` | master | yes | per-user namespace, limits, Kubernetes login |
| `3-test-tenancy.sh <userA> <userB>` | master | yes | test: per-user rules |
| `4-images.sh [<sif>…]` | master (internet) | **no** | serving images |
| `5-test-serving.sh [image]` | master | **no** | test: local serving |
| `6-test-endpoints.sh <user> [ml] [llm] [torch] [--keep]` | master | yes | test: real endpoints as the user |
| `7-install-ood-app.sh` | login node | yes | the Model Hub portal app |
| `8-gateway.sh` | master | yes | the HTTPS gateway |
| `model-register DIR NAME` | any node, as the user (not root) | no | register a Hugging Face folder in MLflow |
| **6-keycloak** | | | |
| `1-install.sh` | master | yes | Keycloak + Postgres under systemd, firewall rule, permanent admin |
| `2-realm.sh` | master | yes | realm, LDAP link, portal client, local admin |
| `kc.py <command>` | master | yes | Keycloak admin calls used by the `1-ldap` scripts |
| **7-jupyter** | | | |
| `1-install-jupyter.sh` | master | yes | own Python + JupyterLab on the shared `/home` |
| `2-install-ood-app.sh` | login node | yes | the Jupyter portal page |
| **8-admin** | | | |
| `1-install.sh` | master | yes | the root helper, sudo rule, log, the page's SSH key |
| `2-install-ood-app.sh` | login node | yes | the Admin page |
| `aistack-admin <action>` | master, as `OOD_ADMIN` with `sudo` | – | see [Admin helper reference](admin-actions.md) |
| **9-monitoring** | | | |
| `1-install.sh` | master | yes | Prometheus, blackbox, node_exporter and collectors on every node |
| `2-install-ood-app.sh` | login node | yes | the Cluster monitor |

## Local checks (no cluster needed)

```bash
bash -n */*.sh
python3 -c 'import ast,sys; [ast.parse(open(f).read()) for f in sys.argv[1:]]' \
  3-mlflow/image/ldap_auth.py 3-mlflow/ood-app/mlflow_k8s/passenger_wsgi.py
python3 9-monitoring/slurm-metrics.py --test
python3 5-model-hub/render.py --test
python3 8-admin/aistack-admin.py --test
```

## Marked shortcuts

Comments starting `# ponytail:` mark deliberate shortcuts with a known limit:

| Where | Shortcut and its limit |
|---|---|
| `1-ldap/add-user.sh` | next user ID = highest + 1; not safe if two admins add people at the same second (the Admin page serializes changes) |
| `1-ldap/add-user.sh` | subuid ranges come from master's files; login nodes stay identical only while this script alone writes them |
| `1-ldap/remove-user.sh` | MLflow only soft-deletes experiments; run `mlflow gc` if it matters |
| `2-ood/branding/_notifications.html.erb` | the bell replaces the portal's Develop menu |
| `3-mlflow/image/ldap_auth.py` | 5-minute login cache; a password change takes up to 5 minutes to bite |
| `4-vscode/gpu-detect.sh` | GPU sizes come from the smallest node; mixed GPU types show as one label |
| `4-vscode/ood-app/vscode/passenger_wsgi.py` | whole-page reload while a session waits or starts |
| `6-keycloak/1-install.sh` | plain HTTP between the login node and Keycloak |
| `8-admin/1-install.sh` | the helper runs scripts from a folder its owner can edit |
| `8-admin/aistack-admin.py` | notices of removed users stay |
| `9-monitoring/…/cluster_status/passenger_wsgi.py` | assumes `gres.conf` lists GPUs in PCI bus order |
