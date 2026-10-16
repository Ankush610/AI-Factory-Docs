# Glossary

Plain meanings of the words used in these pages.

| Word | Plain meaning |
|---|---|
| **`/node` and `/rnode`** | the portal's two ways to pass a browser through to a program on a node. `/rnode` strips its own prefix (VS Code needs that); `/node` keeps it (Jupyter, MLflow) |
| **Admin helper** | the small root program (`aistack-admin`) behind every Admin page button. It accepts only the admin account and a fixed list of actions |
| **Admin notice** | a short note the admin leaves when stopping someone's session, job or endpoint. The user sees it under the portal's bell |
| **Admission policy** | a rule Kubernetes checks before it accepts a new pod. If the pod breaks the rule, it is refused |
| **API** | a way for programs to talk to each other. Here: sending data to a model and getting an answer back |
| **API key** | a long secret string that proves who is calling. One per person in AI-Factory |
| **Apptainer** | a container tool made for HPC. It runs an image as the normal user, with no root and no background service |
| **Artifact** | a file saved with an MLflow run, for example the model itself or a chart |
| **Back-channel logout** | Keycloak telling the portal directly "this login has ended", so open tabs are signed out too |
| **Blackbox probe** | monitoring trying a service the way a user would (open the page, do the TLS handshake) and recording if it answered |
| **CA (certificate authority)** | the "stamp" that makes a certificate trusted. Users download our CA once so their tools trust the gateway |
| **cert-manager** | a Kubernetes add-on that makes and renews certificates |
| **CNI / network plugin** | the part of Kubernetes that connects pods to each other (today: flannel; planned: Calico or Cilium) |
| **code-server** | the open-source build of VS Code that runs on a server and opens in the browser |
| **Container / image** | a ready-made box with software and all its libraries at fixed versions |
| **containerd** | the program on each Kubernetes node that actually runs containers |
| **DeviceClass** | a Kubernetes description of a kind of device, for example "an NVIDIA H100 GPU" |
| **DRA (Dynamic Resource Allocation)** | the newer Kubernetes way to hand GPUs to pods |
| **Endpoint** | a deployed model with an address you can call |
| **Experiment (MLflow)** | a folder of related training runs, named `<user>/<name>` here |
| **flannel** | the current network plugin. Simple, but it can't block traffic between pods |
| **Forced command** | an SSH key that can run only one fixed program, whatever the caller asks. The Admin page's key works this way |
| **Gateway** | the one HTTPS door for calling models. It checks keys and sharing rules |
| **GPU** | the graphics processor used to train and run AI models fast (H100, A30) |
| **Helm** | a package installer for Kubernetes. slurm-bridge is installed with it |
| **htpasswd** | a small password file the web server checks. Only the local admin login uses it, in `ldap` mode |
| **HTTPS / TLS** | encrypted web traffic |
| **Kernel (Jupyter)** | the Python process that runs a notebook's cells. Here, one per training image |
| **Keycloak** | the login server: one login page for every web tool (single sign-on) |
| **Kubeconfig** | the file that tells `kubectl` where the cluster is and who you are. Each user's is `~/.kube/aistack.config` |
| **Kubernetes (k8s)** | the system that runs and looks after long-running services (pods) |
| **LDAP** | the directory that holds the list of users, groups and password hashes |
| **LLM** | large language model, for example Qwen or Llama |
| **Login node (`LOGIN_HOST`)** | the machine users come in through: the portal, SSH, light editor sessions. On the test cluster it is master itself |
| **LoRA** | a small "add-on" trained on top of a big LLM, instead of copying the whole model |
| **Master (`MASTER_HOST`)** | the cluster's "brain": LDAP, Keycloak, MLflow, Slurm controller, Kubernetes control plane, registry, gateway, Prometheus |
| **MetalLB** | gives a Kubernetes service a fixed IP on the local network (used for the gateway) |
| **MLflow** | the tool that records training runs and keeps trained models |
| **Model registry** | MLflow's list of named, numbered models (`churn` v1, v2…) |
| **Namespace** | a private area inside Kubernetes. Each user has one: `u-<user>` |
| **NFS** | the network file system that shares `/home` with every node |
| **Node** | one server in the cluster |
| **node_exporter** | the small program on every node that reports CPU, memory, disk, network and service state to Prometheus |
| **NVLink** | a very fast link between GPUs in the same server, so one model can use 2 GPUs |
| **NVMe** | a very fast local disk inside a server |
| **OCI image** | the standard (Docker-style) container image format Kubernetes uses |
| **OIDC (OpenID Connect)** | the standard way a web app asks a login server "who is this user?" |
| **One-time password** | a first password from the admin that works once; at the first login the user must choose their own |
| **OOD (Open OnDemand)** | the web portal: the front door of AI-Factory |
| **`OOD_ADMIN`** | the one local admin account (`admin`). Not in LDAP, so it still works when LDAP is down. The only user of the Admin page and Cluster monitor |
| **`OOD_AUTH`** | the `site.conf` switch for the portal's login: `keycloak` (one login page) or `ldap` (the browser's own password box) |
| **Open VSX** | the open extension store VS Code in the browser uses instead of Microsoft's |
| **OverSubscribe** | a Slurm setting that lets several jobs share one CPU core. The `viewer` partition allows 4 editor sessions per core |
| **Partition** | a group of nodes in Slurm, each with its own rules |
| **Passenger** | the part of the portal that runs our own app pages (Model Hub, MLflow, Admin…) as the logged-in user |
| **Pickle** | Python's way to save an object (like a trained model) to a file. It needs the same library versions to load |
| **Pod** | the smallest thing Kubernetes runs: one or a few containers together |
| **Podman** | a tool to build and run OCI images, without root |
| **Ponytail comment** | a `# ponytail:` comment in OOD-Install: a deliberate shortcut with a known limit |
| **Prometheus** | the monitoring database on master. It collects numbers from every node every 15 seconds and checks alert rules |
| **PUN** | "per-user nginx": the small web server the portal starts for each logged-in user, running as that user |
| **Quadlet** | a way to run a podman container as a normal systemd service. Keycloak and its database run this way |
| **Quota (ResourceQuota)** | a Kubernetes limit on how much a namespace may use, for example 1 GPU |
| **RBAC** | role-based access control: Kubernetes rules saying who may do what, where |
| **Realm (Keycloak)** | Keycloak's name for one set of users and login settings. Ours is named after the cluster (`aistack`) |
| **Recording rule** | a named Prometheus query, such as `slurm:node_gpus_free`. Apps use the name instead of writing their own query |
| **Registry** | a server that stores container images. Ours is `master:5000` |
| **Reserved vs used** | Slurm *reserves* CPUs, memory and GPUs for a job; the job may *use* less. The Cluster monitor shows both |
| **Restart Web Server** | the portal menu item that restarts your own portal web server, so it picks up changed settings and apps |
| **Rudra** | our single-rack AI system sold to customers. AI-Factory is its software |
| **safetensors** | the safe, common file format for LLM weights |
| **SIF** | an Apptainer image file (`.sif`) |
| **`site.conf`** | the one settings file for a site. Every install script reads it |
| **Slurm** | the job queue: it owns the nodes and GPUs and decides who runs when |
| **slurm-bridge (Slinky)** | the link that turns Kubernetes pods into Slurm jobs, so both share the same GPUs fairly |
| **SSSD** | the program on each node that asks LDAP about users |
| **Subuid / subgid** | a range of extra user IDs a person needs to build images with rootless podman. `add-user.sh` gives one |
| **sudo** | lets one user run one allowed command as root. The Admin page uses it for the admin helper only |
| **Taint** | a "keep out" mark on a Kubernetes node. Only pods that accept it can run there |
| **Tensor parallel** | splitting one model across several GPUs |
| **Textfile collector** | a script that writes numbers into a file that node_exporter then reports. Our GPU and Slurm data arrive this way |
| **Token (MLflow)** | a random password-like string that lets a user's jobs log in to MLflow |
| **Tracking URI** | the MLflow server's address that scripts and jobs use (`MLFLOW_TRACKING_URI`) |
| **UID** | the number Linux uses for a user. The same on every node here, thanks to LDAP |
| **Viewer partition** | the Slurm partition on the login node for light editor sessions |
| **vLLM** | a fast server for running LLMs, with an OpenAI-style API |
| **Widget (OOD)** | a block on the portal's home page. Ours shows the tool cards |
