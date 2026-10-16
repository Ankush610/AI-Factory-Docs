# Problems and fixes

| What you see | Why | What to do |
|---|---|---|
| My new password isn't accepted by MLflow | MLflow remembers logins for up to 5 minutes | wait 5 minutes |
| MLflow: "permission denied" on an experiment | someone else already has an experiment with that name, or it doesn't exist (MLflow gives the same answer for both) | name it `<you>/<name>`, see [Track runs](tracking.md#name-experiments-youname) |
| My job gets 401 from MLflow | your MLflow token is missing or old | ask your admin to renew it |
| My job waits in `squeue` for a long time | the GPUs or memory it asks for are taken | `squeue --me` shows the reason. Ask for less, or wait |
| My job was killed: out of memory | it used more than its `--mem` | ask for more `--mem` |
| My job stopped at the time limit | it ran longer than `--time` | ask for more `--time`, or save checkpoints and continue |
| My VS Code / Jupyter session is gone | its hours ran out, or no browser was attached for 1 hour | start a new one. Your files are still there |
| "Stopped by the admin on …" | the admin stopped it, usually to free a GPU | remove the card; start again when needed |
| My model stays in **Waiting** | all GPUs are busy with training, which comes first | the page shows your place in the queue. Or deploy on CPU |
| My model doesn't show in Model Hub | it isn't registered in MLflow | register it ([Track runs](tracking.md)); for an LLM, run `model-register` |
| Gateway answers 401 | key missing or wrong, or brand new | check `MH_KEY`; a new key works within a few seconds |
| Gateway answers 403 | wrong address, or the endpoint isn't shared with you | check the owner and endpoint name; ask the owner to share it |
| Gateway answers 502 / 504 | the model is stopped, loading or waiting | check its state in **My endpoints** |
| `curl`: certificate problem | the gateway uses the site's own certificate | download `gateway-ca.crt` from the **API** tab and use `--cacert gateway-ca.crt` |
| Jupyter notebook stays "Connecting…" | you opened the portal by a different address than the official one | open the portal by its official address |
| Jupyter `torch.cuda.is_available()` is False | a "Notebook only" session, or a non-container kernel | start a **GPU node** session and pick a `(container)` kernel |
| There is a number on the bell in the top bar | the admin stopped one of your sessions, jobs or endpoints | open the bell to read why; ✕ clears it |
| A VS Code extension can't be found | extensions come from Open VSX, not Microsoft's store | look for an Open VSX alternative |
| Something else | | ask your admin. Tell them what you did, what you saw, and the job number or endpoint name |
