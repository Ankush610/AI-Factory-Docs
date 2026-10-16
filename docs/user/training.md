# Train a model

Training runs as a **Slurm job**: you write a small script that says what you need (GPUs, memory, hours), and
Slurm runs it as soon as the resources are free. Your code runs inside a **ready-made container image** that
already has PyTorch, TensorFlow or classic ML libraries, plus MLflow.

You don't have to keep a browser open. The job keeps running after you log out.

## 1. Pick an image

The images are on shared storage, so every node sees them. List them:

```bash
ls /home/apps/containers/training/*/*.sif
```

| Folder | For |
|---|---|
| `pytorch` | PyTorch deep learning (GPU) |
| `pytorch-llm` | fine-tuning large language models: transformers, PEFT, DeepSpeed |
| `pytorch-nlp`, `pytorch-vision` | PyTorch with text or image libraries |
| `tensorflow*` | TensorFlow / Keras |
| `ml-classic` | scikit-learn, XGBoost, LightGBM (CPU) |

The version is in the file name, for example `pytorch-2.10.0-cuda12.9.sif`. Ask your admin if you need something
that isn't there.

## 2. Write a job script

Save this as `train.sh` next to your `train.py`, and change what you need:

```bash
#!/bin/bash
#SBATCH --job-name=my-training
#SBATCH --partition=standard
#SBATCH --gres=gpu:1               # number of GPUs
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G                  # always set this, see below
#SBATCH --time=04:00:00            # hours:minutes:seconds; the job is stopped after this
#SBATCH --output=logs/%x-%j.out    # %x = job name, %j = job number

source /home/apps/containers/apptainer.sh
SIF=/home/apps/containers/training/pytorch/pytorch-2.10.0-cuda12.9.sif

# MLflow: copy this line from Interactive Apps → MLflow (see "Track runs")
export MLFLOW_TRACKING_URI=<tracking address>
export MLFLOW_EXPERIMENT_NAME="$USER/my-experiment"

apptainer exec --nv "$SIF" python train.py
```

- `--nv` gives the container your job's GPUs.
- **Always set `--mem`.** A job without it gets the whole node's memory, and blocks others on that node.
- For a CPU-only job (classic ML), drop the `--gres` line and use the `ml-classic` image.

Ready-made examples for each image: `/home/apps/containers/test-scripts/training/<family>/run.sh`. Copy one into
your home and change it.

## 3. Submit and follow it

```bash
mkdir -p logs          # Slurm does not create the logs folder for you
sbatch train.sh        # prints: Submitted batch job 1234
```

| To… | Run |
|---|---|
| see your jobs and why they wait | `squeue --me` |
| follow the output | `tail -f logs/my-training-1234.out` |
| cancel a job | `scancel 1234` |
| see what a finished job used | `sacct -j 1234 --format=JobID,State,Elapsed,MaxRSS` |

Run these from the VS Code terminal, a Jupyter terminal, or **Clusters → Shell**.

## Several GPUs or nodes

The images include NCCL and `torchrun` (DeepSpeed in `pytorch-llm`). Ask for more GPUs with `--gres=gpu:N`, and
more nodes with `--nodes=N`. The `run.sh` examples show the `torchrun` setup for several nodes.

## Next

Make your script record its scores and save the model: [Track runs](tracking.md).
