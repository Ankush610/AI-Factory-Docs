# Getting started

## Log in

1. Open the portal address in your browser. Your admin gives it to you, together with your username and a first password.
2. The AI-Factory login page appears. Type your username and password.
3. **First login only:** you are asked to choose your own password. The first password from the admin works once.

The same username and password now work everywhere: the portal, MLflow, Model Hub and SSH. You log in once in
the browser, and every tool knows who you are. A browser login lasts 8 hours.

!!! tip "Signing out"
    Signing out ends the session in every open tab, not only the one you are in.

## The home page

After login you see the home page: the tools as cards, in sections. A card shows only if you may open that tool.

| Section | Cards | Guide |
|---|---|---|
| **Develop** | Jupyter, VS Code, Shell | [Write code](coding.md) |
| **Track & Deploy** | MLflow, Model Hub | [Track runs](tracking.md), [Deploy a model](deploy.md) |
| **Data & Cluster** | Files | below |

The top bar has the same tools in three menus:

- **Files**: your home folder. Browse, upload and download.
- **Clusters**: a terminal in the browser (**Shell**), and the system status.
- **Interactive Apps**: Jupyter, VS Code, MLflow, Model Hub.

## Your files

- Your home folder is `/home/<you>`. It is the **same folder on every node**: the editor, your training jobs and
  your deployed models all see the same files.
- Only you can read it, unless you change the permissions yourself.
- Big files: upload with **Files**, or copy them in over SSH (`scp`, `rsync`).

## Change your password

Open **Clusters → Shell** (or SSH in) and run:

```bash
passwd
```

The new password works everywhere at once. MLflow may take up to 5 minutes to accept it.

## Rules the platform keeps for you

- Everything you start runs **as you**, with your own file permissions. Nobody else's code runs as you.
- **Training comes first.** Deployed models wait for a free GPU. They never stop someone's training job.
- A deployed model runs **1 day** by default, **4 days** at most. You get **1 GPU** for models at a time (the
  admin can raise this to 2).
- An editor or notebook session ends when its hours run out, or after **1 hour** with no browser attached.

## What next

| I want to… | Go to |
|---|---|
| write code or notebooks | [Write code](coding.md) |
| train on a GPU | [Train a model](training.md) |
| record runs and keep models | [Track runs](tracking.md) |
| put a model online | [Deploy a model](deploy.md) |
| call a model from code, share it | [Call and share a model](call-api.md) |
| fix something that doesn't work | [Problems and fixes](faq.md) |
