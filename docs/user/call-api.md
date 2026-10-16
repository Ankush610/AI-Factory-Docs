# Call and share a model

Every deployed model is called through **one HTTPS address**, the **gateway**, with your **personal API key**.
This works from a notebook on the cluster and from your laptop on the site's network.

```
https://<gateway>/<owner>/<endpoint>/<path>
```

## 1. Make your API key

1. Model Hub → **API key** → make a new key.
2. **Copy it now. It is shown only once.** It looks like `mh~alice~…`.
3. Keep it in an environment variable, not in your code:

    ```bash
    export MH_KEY='mh~alice~…'
    ```

One key works for every model you may call: yours, and those shared with you. Lost it, or it leaked? Make a
new one. The old key stops working within about a minute.

## 2. Get the certificate (if Model Hub offers it)

Many sites use their own certificate for the gateway. Then the **API** tab has a **Download certificate** button.
Download `gateway-ca.crt` once. It is the same for everyone. The examples below use it.

## 3. Call it

The endpoint's **API** tab has these examples filled in for you, ready to copy.

=== "curl"

    ```bash
    curl --cacert gateway-ca.crt https://<gateway>/alice/churn/invocations \
      -H "Authorization: Bearer $MH_KEY" \
      -H 'Content-Type: application/json' \
      -d '{"dataframe_records": [{"age": 42, "plan": "gold"}]}'
    ```

=== "Python (ML / DL)"

    ```python
    import os, requests

    r = requests.post("https://<gateway>/alice/churn/invocations",
                      json={"dataframe_records": [{"age": 42, "plan": "gold"}]},
                      headers={"Authorization": "Bearer " + os.environ["MH_KEY"]},
                      verify="gateway-ca.crt")
    print(r.json())
    ```

=== "Python (LLM, OpenAI client)"

    ```python
    import os, httpx
    from openai import OpenAI

    client = OpenAI(base_url="https://<gateway>/alice/qwen-legal/v1",
                    api_key=os.environ["MH_KEY"],
                    http_client=httpx.Client(verify="gateway-ca.crt"))
    reply = client.chat.completions.create(
        model="qwen-legal",
        messages=[{"role": "user", "content": "Hello!"}])
    print(reply.choices[0].message.content)
    ```

Any tool that speaks the OpenAI API works with an LLM endpoint: give it the `base_url` and your key. The exact
model name to use is on the endpoint's **API** tab.

## Share an endpoint

On your endpoint, click **Share** and pick who may call it:

| Choice | Who may call it |
|---|---|
| Only me | you (the default) |
| People | you and the users you name |
| Teams | you and the members of the teams you pick (the admin makes teams) |
| Everyone | any user with a key |

- Each person calls it with **their own** key. You never give your key away.
- They find it in Model Hub under **My endpoints → Shared with me**.
- You can change or take back the sharing at any time. Deleting the endpoint deletes its sharing too.

## What the answer codes mean

| Code | Means | Do |
|---|---|---|
| 200 | OK | |
| 401 | missing, wrong or old key | check `MH_KEY`. A new key needs up to a minute to work |
| 403 | no such endpoint, **or** it isn't shared with you | check the address; ask the owner to share it |
| 502 / 504 | the model is stopped, still loading, or waiting for a GPU | look at its state in **My endpoints** |

403 is the same answer for both cases on purpose, so nobody can guess the names of other people's endpoints.
