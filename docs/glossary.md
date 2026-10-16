# Glossary

Plain meanings of the words used in these pages.

| Word | Plain meaning |
|---|---|
| **Admin helper** | the small root program (`aistack-admin`) behind every Admin page button. It accepts only the admin account and a fixed list of actions |
| **Admission policy** | a rule Kubernetes checks before it accepts a new pod. If the pod breaks the rule, it is refused |
| **Apptainer** | a container tool made for HPC. It runs an image as the normal user, with no root and no background service |
| **API** | a way for programs to talk to each other. Here: sending data to a model and getting an answer back |
| **API key** | a long secret string that proves who is calling. One per person in AI-Factory |
| **Artifact** | a file saved with an MLflow run, for example the model itself or a chart |
| **Back-channel logout** | Keycloak telling the portal directly "this login has ended", so open tabs are signed out too |
| **CA (certificate authority)** | the "stamp" that makes a certificate trusted. Users download our CA once so their tools trust the gateway |
| **cert-manager** | a Kubernetes add-on that makes and renews certificates |
| **CNI / network plugin** | the part of Kubernetes that connects pods to each other (today: flannel; planned: Calico or Cilium) |
| **Container / image** | a ready-made box with software and all its libraries at fixed versions |
| **containerd** | the program on each Kubernetes node that actually runs containers |
| **DeviceClass** | a Kubernetes description of a kind of device, for example "an NVIDIA H100 GPU" |
| **DRA (Dynamic Resource Allocation)** | the newer Kubernetes way to hand GPUs to pods |
| **Endpoint** | a deployed model with an address you can call |
| **Experiment (MLflow)** | a folder of related training runs, named `<user>/<name>` here |
| **flannel** | the current network plugin. Simple, but it can't block traffic between pods |
| **Gateway** | the one HTTPS door for calling models. It checks keys and sharing rules |
| **GPU** | the graphics processor used to train and run AI models fast (H100, A30) |
| **Helm** | a package installer for Kubernetes. slurm-bridge is installed with it |
| **HTTPS / TLS** | encrypted web traffic |
| **Kernel (Jupyter)** | the Python process that runs a notebook's cells. Here, one per training image |
| **Keycloak** | the login server: one login page for every web tool (single sign-on) |
| **Kubernetes (k8s)** | the system that runs and looks after long-running services (pods) |
| **LDAP** | the directory that holds the list of users, groups and password hashes |
| **LLM** | large language model, for example Qwen or Llama |
| **LoRA** | a small "add-on" trained on top of a big LLM, instead of copying the whole model |
| **MetalLB** | gives a Kubernetes service a fixed IP on the local network (used for the gateway) |
| **MLflow** | the tool that records training runs and keeps trained models |
| **Model registry** | MLflow's list of named, numbered models (`churn` v1, v2…) |
| **Namespace** | a private area inside Kubernetes. Each user has one: `u-<user>` |
| **NFS** | the network file system that shares `/home` with every node |
| **Node** | one server in the cluster |
| **NVLink** | a very fast link between GPUs in the same server, so one model can use 2 GPUs |
| **NVMe** | a very fast local disk inside a server |
| **OCI image** | the standard (Docker-style) container image format Kubernetes uses |
| **OIDC (OpenID Connect)** | the standard way a web app asks a login server "who is this user?" |
| **One-time password** | a first password from the admin that works once; at the first login the user must choose their own |
| **OOD (Open OnDemand)** | the web portal: the front door of AI-Factory |
| **Partition** | a group of nodes in Slurm, each with its own rules |
| **Pickle** | Python's way to save an object (like a trained model) to a file. It needs the same library versions to load |
| **Pod** | the smallest thing Kubernetes runs: one or a few containers together |
| **Podman** | a tool to build and run OCI images, without root |
| **Quota (ResourceQuota)** | a Kubernetes limit on how much a namespace may use, for example 1 GPU |
| **RBAC** | role-based access control: Kubernetes rules saying who may do what, where |
| **Registry** | a server that stores container images. Ours is `master:5000` |
| **Rudra** | our single-rack AI system sold to customers. AI-Factory is its software |
| **safetensors** | the safe, common file format for LLM weights |
| **SIF** | an Apptainer image file (`.sif`) |
| **`site.conf`** | the one settings file for a site. Every install script reads it |
| **Slurm** | the job queue: it owns the nodes and GPUs and decides who runs when |
| **slurm-bridge (Slinky)** | the link that turns Kubernetes pods into Slurm jobs, so both share the same GPUs fairly |
| **SSSD** | the program on each node that asks LDAP about users |
| **sudo** | lets one user run one allowed command as root. The Admin page uses it for the admin helper only |
| **Taint** | a "keep out" mark on a Kubernetes node. Only pods that accept it can run there |
| **Tensor parallel** | splitting one model across several GPUs |
| **Token (MLflow)** | a random password-like string that lets a user's jobs log in to MLflow |
| **UID** | the number Linux uses for a user. The same on every node here, thanks to LDAP |
| **vLLM** | a fast server for running LLMs, with an OpenAI-style API |
| **Viewer partition** | the Slurm partition on the login node for light editor sessions |
| **Widget (OOD)** | a block on the portal's home page. Ours shows the tool cards |
