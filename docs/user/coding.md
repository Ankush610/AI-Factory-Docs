# Write code: VS Code and Jupyter

Two coding tools open in your browser, from the portal:

- **VS Code**: a full code editor with a terminal.
- **JupyterLab**: notebooks.

Each session runs **on the cluster, as you**, next to your files and the GPUs. Nothing to install on your laptop.

## Start a session

1. **Interactive Apps → VS Code** (or **Jupyter**).
2. Pick where it runs (see below) and how many hours.
3. Click **Launch**. After a few seconds the card shows **Running**.
4. Click **Connect**. The editor or notebook opens in a new tab.

| Choice | Runs on | You get | Use it for |
|---|---|---|---|
| **Editor only** / **Notebook only** | the login node | 1 shared CPU, 2 GB (VS Code) or 4 GB (Jupyter) | writing code, small tests |
| **GPU node: n × &lt;GPU&gt;** | a GPU node | n GPUs, plus a matching share of that node's CPUs and memory | trying things on real GPUs |

!!! warning "A GPU session holds its GPUs"
    A GPU session keeps its GPUs until it ends, even when you are not using them. For long training, submit a
    [training job](training.md) instead. A job gives the GPUs back the moment it finishes.

## Jupyter kernels

Every training image on the cluster shows up as a ready kernel, named `<image> (container)`: PyTorch, PyTorch-LLM,
classic ML and others. Pick one and its libraries are there, with GPU support. A new image the admin adds shows
up in your next session.

You can also add your own Python environment as a kernel, the usual way:

```bash
python -m ipykernel install --user --name my-env
```

## When a session ends

- When its hours run out.
- After **1 hour with no browser attached** (and, in Jupyter, no cell running). This frees memory and GPUs for others.
- When the admin stops it. The page then shows **"Stopped by the admin on &lt;time&gt;"**. Remove that card with its bin button.

Your files are not lost when a session ends. Only the running programs stop.

**Stop** on the card ends a session early. Please do this when you are done with a GPU session.

## VS Code extensions

Extensions come from **Open VSX**, not Microsoft's store. Python, Jupyter, GitLens and most others are there.
A few Microsoft-only ones (Pylance, Copilot, Remote-SSH) are not.

Your settings and extensions are kept in your home folder, so they are there next time.
