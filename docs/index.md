# AI-Factory

AI-Factory turns a rack of servers into a **shared AI platform**. You open a web page, log in once, and can then:

- write code and notebooks in the browser,
- train models on GPUs,
- keep a record of every training run,
- put a trained model online as an API,
- let colleagues (or everyone) call that API.

You don't need root, and you don't need to know Slurm or Kubernetes.

It is built for **Rudra**, a single-rack AI system. Every Rudra gets the same software, installed by our scripts.

## Pick your path

<div class="grid cards" markdown>

-   :material-account: **I use the platform**

    ---

    Log in, write code, train, track runs, deploy a model and call it.

    [:octicons-arrow-right-24: User guide](user/index.md)

-   :material-shield-account: **I run the platform**

    ---

    Install it, add people, set limits, watch the cluster, fix problems.

    [:octicons-arrow-right-24: Admin guide](admin/index.md)

-   :material-cog: **I want to know how it works**

    ---

    Each part explained, with diagrams: login, portal, Slurm, Kubernetes, MLflow, Model Hub, gateway.

    [:octicons-arrow-right-24: How it works](how-it-works/index.md)

-   :material-book-alphabet: **What does that word mean?**

    ---

    Plain meanings of the technical words used on these pages.

    [:octicons-arrow-right-24: Glossary](glossary.md)

</div>

## A user's path in one picture

```mermaid
flowchart LR
    A["Log in"] --> B["Write code<br/>VS Code / Jupyter"]
    B --> C["Train<br/>Slurm job on a GPU"]
    C --> D["Track + register<br/>MLflow"]
    D --> E["Deploy<br/>Model Hub"]
    E --> F["Call it<br/>gateway + API key"]
    F --> G["Share it"]
```

## Where the code lives

| Repository | What is in it |
|---|---|
| [OOD-Install](https://github.com/Ankush610/OOD-Install) | the install scripts, one numbered folder per component, and `site.conf` with every site value |
| [OOD-Container](https://github.com/Ankush610/OOD-Container) | the Apptainer recipes for the training and serving images |
| [AI-Factory-Docs](https://github.com/Ankush610/AI-Factory-Docs) | these pages |
