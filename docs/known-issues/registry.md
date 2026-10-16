# Registry

Issues in [`0-registry`](https://github.com/Ankush610/OOD-Install/tree/main/0-registry). How to read this page:
[Known issues](index.md#how-to-read-these-pages).

| Issue | Type | Confidence |
|---|---|---|
| [No login, deletes on, no firewall rule](#no-login-deletes-on-no-firewall-rule) | Security | Confirmed |
| [`k8s-image mirror` defaults to `:latest`](#k8s-image-mirror-defaults-to-latest) | Bug | Confirmed |
| [`k8s-image` help shows `master:5000`](#k8s-image-help-shows-master5000) | Docs | Confirmed |
| [Registry image not pinned](#registry-image-not-pinned) | Gap | Confirmed |
| [Garbage collection runs on the live registry](#garbage-collection-runs-on-the-live-registry) | Docs | Inferred |
| [Uneven node coverage with a separate login node](#uneven-node-coverage-with-a-separate-login-node) | Gap | Confirmed |
| [`k8s-image ls` may cut the list short](#k8s-image-ls-may-cut-the-list-short) | Bug | Inferred |

## No login, deletes on, no firewall rule

- **Type:** Security · **Confidence:** Confirmed · **Where:** `0-registry/setup-registry.sh:41-43`
- **What happens:** the registry has no login and no TLS, and deletes are turned on. Anyone who can reach port
  5000 on master can push, overwrite or delete any image, including MLflow's and Keycloak's. The script opens or
  restricts no firewall port; the README only says to open 5000 for `LOGIN_IP`.
- **Workaround:** allow port 5000 on master only from the cluster nodes and `LOGIN_IP`, for example with a
  firewalld rich rule.

## `k8s-image mirror` defaults to `:latest`

- **Type:** Bug · **Confidence:** Confirmed · **Where:** `0-registry/k8s-image:78`, help example at `:19-20`
- **What happens:** `k8s-image mirror SRC` with no tag pushes `:latest`. The help example even mirrors
  `vllm-openai:latest`. This goes against the "never `:latest`" rule: nodes cache images, so a moved tag runs
  different code on different nodes.
- **Workaround:** always give both tags: `k8s-image mirror docker.io/x/y:1.2 y:1.2`.

## `k8s-image` help shows `master:5000`

- **Type:** Docs · **Confidence:** Confirmed · **Where:** `0-registry/k8s-image:3-33`
- **What happens:** only `@REGISTRY@` is filled in at install. The help text printed by `k8s-image help` says
  `master:5000` literally, so it is wrong on a site with another master name or port. The commands still work.

## Registry image not pinned

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `site.conf:16`
- **What happens:** `REGISTRY_IMAGE=docker.io/library/registry:2` is a floating major tag. Every other image is
  pinned to an exact version.
- **Workaround:** pin a patch version or digest in `site.conf`.

## Garbage collection runs on the live registry

- **Type:** Docs · **Confidence:** Inferred (from upstream registry documentation, not from the repo) ·
  **Where:** `0-registry/README.md:47`
- **What happens:** the README runs `registry garbage-collect` on the running container. Upstream advises read-only
  mode or a stopped registry during garbage collection, because a push during it can lose layers.
- **Workaround:** run it when nobody is pushing, or stop pushes first.

## Uneven node coverage with a separate login node

- **Type:** Gap · **Confidence:** Confirmed · **Where:** `0-registry/setup-registry.sh:22-23`
- **What happens:** containerd is set up on master + `COMPUTE_NODES`, podman on master + `LOGIN_NODES`. So:
    - a separate login node that is also a Kubernetes node gets no containerd config;
    - compute nodes get no podman trust and no `k8s-image` on `PATH`.
- **Open question:** is podman on compute nodes expected to work with the registry?

## `k8s-image ls` may cut the list short

- **Type:** Bug · **Confidence:** Inferred · **Where:** `0-registry/k8s-image:87`
- **What happens:** it reads `/v2/_catalog` once with no paging. With many repositories the list may be truncated.
