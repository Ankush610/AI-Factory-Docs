# Track runs and keep models (MLflow)

**MLflow** is your lab notebook and model shelf:

- **Tracking:** every training run records its settings, scores, charts and files.
- **Model registry:** the runs worth keeping become named, numbered models (`churn` v1, v2…).

There is one MLflow for everyone, but **you see only your own** experiments and models. Model Hub reads its list
of models from MLflow, so **every model you want to deploy goes through MLflow first**.

## Set it up once

Open **Interactive Apps → MLflow**. The page shows your **tracking address** with a **Copy** button. Put that
line in your job script (or run it in your terminal):

```bash
export MLFLOW_TRACKING_URI=<tracking address>
```

You don't need a password in your scripts. Your personal MLflow token is already in `~/.mlflow/credentials`, and
MLflow reads it by itself. Keep that file private.

## Name experiments `<you>/<name>`

Experiment names are shared by everyone. If someone else already has `mnist`, you get "permission denied". So
always start the name with your username:

```python
mlflow.set_experiment("alice/mnist")
```

## Log a run

```python
import mlflow

mlflow.set_experiment("alice/churn")
with mlflow.start_run():
    mlflow.log_param("lr", 0.001)
    mlflow.log_metric("accuracy", 0.94)
    mlflow.sklearn.log_model(model, "model", registered_model_name="churn")
```

- Use the logger for your library: `mlflow.sklearn`, `mlflow.pytorch`, `mlflow.xgboost`…
- `registered_model_name=` registers it straight away as a new version of `churn`. Or leave it out, and register
  the best run later in the MLflow web page: open the run, then **Register model**.

## Compare runs

Click **Open MLflow** on the MLflow page. Pick an experiment, tick some runs, and click **Compare** to see their
settings and scores side by side.

## Large language models

An LLM is many gigabytes, so it is **not** copied into MLflow. Its files stay in your home, and MLflow only keeps
a record that points to them:

1. Save the model with `save_pretrained` into a folder under `~/models/`. It must have `config.json` and
   `*.safetensors`. A LoRA adapter must be merged into the model first.
2. Register it:

    ```bash
    model-register ~/models/my-llm/v1 my-llm
    ```

3. It now shows in Model Hub like any other model. `model-register --help` explains the options, including an
   optional `MODEL.yaml` with a description and license.

## Next

[Deploy the model](deploy.md) as an API.
