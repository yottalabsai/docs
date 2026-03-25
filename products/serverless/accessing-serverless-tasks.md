# Accessing Serverless Tasks

{% hint style="info" %}
Only tasks in the **Running** state can be accessed.
{% endhint %}

* Click the **Running** card to open the detailed page and find the endpoint URL.

<figure><img src="../../.gitbook/assets/image (178).png" alt="" width="563"><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (182).png" alt="" width="563"><figcaption></figcaption></figure>

Example: Access via `curl`

{% hint style="warning" %}
Each request must include the header: `Authorization: Bearer <YOUR_API_KEY>`\
API keys can be found in **Settings → Access Keys**.

See more details in [API keys doc](https://docs.yottalabs.ai/yotta-labs/api-and-sdk/api-keys)
{% endhint %}

```bash
curl --location 'https://2wmczkcst63e.yottadeos.com/v1/chat/completions' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer <YOUR_API_KEY>' \
--data '{
  "temperature": 0.5,
  "top_p": 0.9,
  "max_tokens": 256,
  "frequency_penalty": 0.3,
  "presence_penalty": 0.2,
  "repetition_penalty": 1.2,
  "model": "meta-llama/Llama-3.2-3B-Instruct",
  "messages": [
    {
      "role": "user",
      "content": "Explain what is AI infrastructure."
    }
  ],
  "stream": false
}'
```

{% hint style="info" %}
The base URL is unique to your task (e.g. `https://2t5y1srid9mo.yottadeos.com`). Replace the `localhost` in the URL of your local deployment to get the full URL (e.g. in the above example, it would be `https://2t5y1srid9mo.yottadeos.com/v1/chat/completions`) of your service managed by the serverless task.
{% endhint %}

