# 0 · Container registry

> One private image registry on master, so images built with podman reach Kubernetes' containerd on every node
> over the cluster LAN.

**Folder:** [`OOD-Install/0-registry`](https://github.com/Ankush610/OOD-Install/tree/main/0-registry) ·
**Explained in:** [Containers and registry](../../how-it-works/06-containers-and-registry.md) ·
**Known issues:** [Registry](../../known-issues/registry.md)

## What it sets up

- A stock `registry:2` container on master at `REGISTRY` (`master:5000`), run by root's podman (not by Kubernetes),
  with deletes turned on. `podman-restart.service` starts it again after a reboot.
- containerd on every Kubernetes node (master + `COMPUTE_NODES`) set to pull from it over plain HTTP.
- podman on master and `LOGIN_NODES` set to trust it, so nobody needs `--tls-verify=false`.
- The `k8s-image` tool in `/home/apps/bin`, on every user's `PATH`, to build, push, mirror and list images without sudo.

It runs first, because MLflow, Model Hub and Keycloak push their images here.

## Before you start

| | |
|---|---|
| **Runs on** | master, as root. It refuses any other host |
| **Needs** | root SSH from master to every node in `COMPUTE_NODES` and `LOGIN_NODES` (unreachable nodes are skipped and listed at the end) |
| | `podman` on master and the login nodes; `containerd` + `crictl` on every Kubernetes node |
| | internet on master once, to pull `REGISTRY_IMAGE` |
| | if the login node is separate: port 5000 on master open for `LOGIN_IP` (the script opens no port) |
| **Side effect** | if master's containerd config changes, restarting containerd bounces the control-plane pods for about a minute |

## Settings used

| Variable | Example | Meaning |
|---|---|---|
| `REGISTRY` | `$MASTER_HOST:5000` | Registry address. Must be `$MASTER_HOST:<port>`, or the script refuses |
| `REGISTRY_DATA` | `/home/apps/registry` | Image storage, mounted at `/var/lib/registry` in the container |
| `REGISTRY_IMAGE` | `docker.io/library/registry:2` | The registry server image (also used as the self-test image) |
| `COMPUTE_NODES`, `LOGIN_NODES` | `cn01 cn02`, `$LOGIN_HOST` | Which nodes get containerd and podman settings |

## Install

```bash
# on master
sudo bash 0-registry/setup-registry.sh
```

| Step | What it does | Expect |
|---|---|---|
| `== 1. Registry container` | Creates `REGISTRY_DATA`. Recreates the container only if its image, mount, delete setting or port differ from `site.conf` (data stays). Enables `podman-restart.service`. Waits up to 20 s for `/v2/` | `running as wanted` or `started …registry:2` |
| `== 2. containerd pulls from REGISTRY` | On master + `COMPUTE_NODES`: sets containerd's `config_path` to `/etc/containerd/certs.d` (backing up `config.toml` first), writes `certs.d/<REGISTRY>/hosts.toml`. Restarts containerd only where something changed | `already configured` or `configured, containerd restarted` per node |
| `== 3. podman trusts REGISTRY` | On master + `LOGIN_NODES`: writes `/etc/containers/registries.conf.d/010-local-registry.conf` (`insecure = true`) and `/etc/profile.d/apps-bin.sh` (adds `/home/apps/bin` to `PATH`) | one line per node |
| `== 4. k8s-image` | Installs `/home/apps/bin/k8s-image` with the registry address filled in, then lists the registry | the registry's contents |
| `== 5. Check` | Pushes a self-test image, pulls it with `crictl` on every Kubernetes node, then deletes it | `<node>: pulled ok` for every node, then `Done.` |

If a node was missed, the last line says `NOT done on: <nodes>. Rerun this script when they're back.`

## Check it works

```bash
# on master
curl -s http://master:5000/v2/_catalog          # JSON list of repositories
sudo podman ps --filter name=registry           # the container is Up
systemctl is-enabled podman-restart.service     # enabled

# on any Kubernetes node, as root
cat /etc/containerd/certs.d/master:5000/hosts.toml
```

Users need a **new login shell** before `k8s-image` is on their `PATH`.

## Everyday tasks

### Push an image (any user, no sudo)

```bash
# on master or a login node, as yourself
k8s-image build myapp:v1 .                       # build + push → master:5000/myapp:v1
k8s-image mirror docker.io/library/redis:7       # copy an upstream image in
k8s-image ls                                     # list everything in the registry
```

In a Kubernetes manifest, write `image: master:5000/<name>:<tag>`. **Always use a fixed tag** (`:v1`), never `:latest`:
each node caches images, so a moved tag means each node runs whatever copy it cached.

| Command | What it does |
|---|---|
| `k8s-image build NAME:TAG [DIR] [-f FILE]` | `podman build` from `DIR` (default `.`), then push. The tag is required |
| `k8s-image NAME:TAG [DIR]` | short for `build` |
| `k8s-image push NAME:TAG` | push an image you already tagged as `REGISTRY/NAME:TAG` |
| `k8s-image mirror SRC [NAME:TAG]` | pull `SRC`, tag it, push it. Give a tag: without one it uses `:latest` |
| `k8s-image ls` | list `repo:tag` for every image |

### Delete an image (root, on master)

Deleting by digest removes **every** tag that shares that digest. Check which tags you keep first.

```bash
d=$(curl -sI -H 'Accept: application/vnd.oci.image.manifest.v1+json' \
      -H 'Accept: application/vnd.docker.distribution.manifest.v2+json' \
      http://$REGISTRY/v2/<name>/manifests/<tag> \
    | awk -F': ' 'tolower($1)=="docker-content-digest"{print $2}' | tr -d '\r')
curl -X DELETE http://$REGISTRY/v2/<name>/manifests/$d          # expect 202
sudo podman exec registry registry garbage-collect --delete-untagged /etc/docker/registry/config.yml
```

!!! warning
    Avoid pushes while garbage collection runs. See [known issues](../../known-issues/registry.md#garbage-collection-runs-on-the-live-registry).

### Other tasks

| Task | How |
|---|---|
| Add a compute or login node | add it to `COMPUTE_NODES` / `LOGIN_NODES`, rerun `setup-registry.sh`. Only the new node changes |
| A node was down during install | rerun the script when it is back |
| Restart or check the registry | `podman ps --filter name=registry`, `podman restart registry`, `curl -sf http://master:5000/v2/ && echo up` |
| Change port, storage or image | edit `REGISTRY`, `REGISTRY_DATA` or `REGISTRY_IMAGE`, rerun. Changing `REGISTRY` renames every image, so push them again |
| Back up | `REGISTRY_DATA`, not the node caches |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Pod `ErrImagePull` / `server gave HTTP response to HTTPS client` | that node's containerd has no `hosts.toml` for the registry | rerun `setup-registry.sh` (the node was probably down) |
| `podman push`: `server gave HTTP response to HTTPS client` | no podman drop-in on that node | rerun; or the user is on a node not in `LOGIN_NODES` |
| `podman push` as a user: `cannot find UID/GID … subuid` | LDAP user without a subuid range | `1-ldap/add-user.sh` gives one; for older users add `user:start:65536` to `/etc/subuid` and `/etc/subgid` |
| Everything `ErrImagePull` after master rebooted | registry container not started at boot | `systemctl enable --now podman-restart.service` (step 1 does this) |
| `DELETE` answers `405` | registry started without deletes enabled | rerun `setup-registry.sh` (it recreates the container and keeps the data) |
| `no registry config_path in config.toml` | containerd config has no `config_path` lines | add `config_path = '/etc/containerd/certs.d'` under the CRI images registry section, rerun |
| `REGISTRY (…) must be master:<port>` | `REGISTRY` points to another host | set `REGISTRY=$MASTER_HOST:<port>` |
| `registry doesn't answer on http://…/v2/` | container failed, port taken, or the name doesn't resolve | `podman logs registry`; check `MASTER_HOST` in `/etc/hosts`; check the port is free |
| `<node>: SKIPPED` | no root SSH to that node, or it is down (5 s timeout) | fix SSH or the node, rerun |
| `<node>: PULL FAILED (crictl pull …)` | containerd there can't reach or doesn't trust the registry | check `hosts.toml`, `config_path`, and the network to port 5000; rerun |
| `k8s-image: ls: cannot reach master:5000` | registry down or port blocked | check the registry on master |
| `k8s-image: command not found` | `PATH` not updated in this shell, or node not in master/`LOGIN_NODES` | open a new login shell, or run `/home/apps/bin/k8s-image` |
| Alert `ServiceUnreachable` with `service=registry` | the monitoring probe of `/v2/` fails | `curl http://master:5000/v2/`, `podman ps` |
