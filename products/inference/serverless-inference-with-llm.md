# Serverless Inference with LLM

This tutorial walks you through deploying an open-source LLM (Qwen2.5-7B-Instruct) as a serverless inference endpoint on YottaLabs using vLLM. By the end, you will have a live OpenAI-compatible API endpoint running on a GPU in the cloud.

## What You Will Build

A serverless vLLM endpoint on YottaLabs that:

* Runs **Qwen2.5-7B-Instruct** on an NVIDIA RTX 5090 (32 GB VRAM)
* Exposes an **OpenAI-compatible** `/v1/chat/completions` API
* Scales workers on demand via the YottaLabs platform
* Can be called from any HTTP client or the OpenAI Python SDK

## Prerequisites

* A YottaLabs account with an API key (`x-api-key`)
* `curl` installed locally (or any HTTP client)
* For gated models (e.g. Llama 3.1): a Hugging Face token with approved access. Qwen2.5 is fully open and requires no token.

## Architecture Overview

```
Your Client
    │
    │  POST /v2/serverless/{id}/tasks
    ▼
YottaLabs Platform  ──────────────────────────────────────────────────┐
    │                                                                  │
    │  Routes request to an available worker                          │
    ▼                                                                  │
Worker (RTX 5090 GPU)                                                  │
    │                                                                  │
    │  vLLM serving Qwen2.5-7B-Instruct                               │
    │  OpenAI-compatible API on port 8000                             │
    │                                                                  │
    └──────────────────────────────────────────────────────────────────┘
```

YottaLabs Serverless has two service modes relevant to LLM inference:

* **ALB mode**: Requests are proxied directly to the worker in real time. Best for synchronous, low-latency calls.
* **QUEUE mode**: Requests are queued and results are returned asynchronously (via polling or webhook). Best for long-running or batch tasks.

This tutorial uses **QUEUE mode**, which is the standard mode for the Serverless v2 API.

{% stepper %}
{% step %}
### Create a Serverless Endpoint

Send a `POST` request to `/v2/serverless` to create your endpoint. This provisions a worker container running `vllm/vllm-openai:latest` and starts the vLLM server.

```bash
curl --http1.1 -X POST https://api.yottalabs.ai/v2/serverless \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "vllm-qwen25-7b",
    "image": "vllm/vllm-openai:latest",
    "serviceMode": "QUEUE",
    "workers": 1,
    "containerVolumeInGb": 50,
    "resources": [
      {
        "region": "us-east",
        "gpuType": "NVIDIA_RTX_5090_32G",
        "gpuCount": 1
      }
    ],
    "expose": {
      "port": 8000,
      "protocol": "http"
    },
    "initializationCommand": "vllm serve Qwen/Qwen2.5-7B-Instruct --host 0.0.0.0 --port 8000 --gpu-memory-utilization 0.90"
  }'
```

**Key parameters explained:**

| Parameter                | Value                     | Notes                                                       |
| ------------------------ | ------------------------- | ----------------------------------------------------------- |
| `image`                  | `vllm/vllm-openai:latest` | Official vLLM Docker image with OpenAI-compatible server    |
| `serviceMode`            | `QUEUE`                   | Async task queue mode for serverless inference              |
| `gpuType`                | `NVIDIA_RTX_5090_32G`     | 32 GB VRAM, sufficient for 7B models in bf16                |
| `containerVolumeInGb`    | `50`                      | Disk space for model weights download (\~15 GB for 7B)      |
| `initializationCommand`  | `vllm serve ...`          | Starts the vLLM OpenAI server on port 8000                  |
| `gpu-memory-utilization` | `0.90`                    | Use 90% of VRAM for the KV cache; leave headroom for the OS |

{% hint style="info" %}
Use `--http1.1` with curl to avoid HTTP/2 protocol errors on some network configurations.
{% endhint %}

**Example response:**

```json
{
  "code": 10000,
  "message": "success",
  "data": {
    "id": 12345,
    "name": "vllm-qwen25-7b",
    "status": "INITIALIZING",
    "serviceMode": "QUEUE",
    "image": "vllm/vllm-openai:latest"
  }
}
```

Save the `id` field — you will need it for all subsequent requests.
{% endstep %}

{% step %}
### Wait for the Endpoint to Be Ready

The worker needs time to pull the Docker image and download model weights from Hugging Face. Poll the endpoint status until it shows `RUNNING`.

```bash
curl --http1.1 https://api.yottalabs.ai/v2/serverless \
  -H "x-api-key: YOUR_API_KEY"
```

Look for `"status": "RUNNING"` in the response for your endpoint. This typically takes **3–8 minutes** on first launch (model weights are \~15 GB). Subsequent starts are faster if weights are cached.

You can also monitor the worker logs from the YottaLabs console. A successful startup looks like this:

```
INFO: Route: /v1/chat/completions, Methods: POST
INFO: Route: /v1/completions, Methods: POST
INFO: Route: /v1/models, Methods: GET
INFO: Application startup complete.
```
{% endstep %}

{% step %}
### Call the Inference Endpoint

Once the endpoint is `RUNNING`, you will find its domain in the endpoint details (e.g. `39q67x443xvy.yottadeos.com`). Call it directly using your YottaLabs API key as a Bearer token:

```bash
curl --http1.1 -X POST https://YOUR_ENDPOINT_DOMAIN/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "Qwen/Qwen2.5-7B-Instruct",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Explain what vLLM is in two sentences."}
    ],
    "max_tokens": 256,
    "temperature": 0.7
  }'
```

{% hint style="info" %}
The endpoint domain sits behind a YottaLabs gateway. Use `Authorization: Bearer YOUR_API_KEY` — the same key used for `x-api-key` on the platform API.
{% endhint %}

**Example response:**

```json
{
  "id": "chatcmpl-1f2af1e9d880fb3a08712b428e176a85",
  "object": "chat.completion",
  "model": "Qwen/Qwen2.5-7B-Instruct",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "vLLM is an open-source library designed to accelerate large language models by improving inference speed and efficiency on GPUs. It supports popular frameworks like Hugging Face Transformers and aims to make LLMs more accessible for real-time applications."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 30,
    "completion_tokens": 63,
    "total_tokens": 93
  }
}
```

The response is a standard OpenAI chat completion object, fully compatible with any tool in the OpenAI ecosystem.
{% endstep %}

{% step %}
### Using the OpenAI Python SDK

Call your endpoint with the OpenAI SDK by overriding `base_url` and passing your API key:

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://YOUR_ENDPOINT_DOMAIN/v1",
    api_key="YOUR_API_KEY",  # YottaLabs API key, used as Bearer token
)

response = client.chat.completions.create(
    model="Qwen/Qwen2.5-7B-Instruct",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is PagedAttention?"},
    ],
    max_tokens=256,
    temperature=0.7,
)

print(response.choices[0].message.content)
```
{% endstep %}
{% endstepper %}

## Managing Your Endpoint

**List all endpoints:**

```bash
curl --http1.1 https://api.yottalabs.ai/v2/serverless \
  -H "x-api-key: YOUR_API_KEY"
```

**Stop an endpoint (pause billing):**

```bash
curl --http1.1 -X POST https://api.yottalabs.ai/v2/serverless/12345/stop \
  -H "x-api-key: YOUR_API_KEY"
```

**Restart a stopped endpoint:**

```bash
curl --http1.1 -X POST https://api.yottalabs.ai/v2/serverless/12345/start \
  -H "x-api-key: YOUR_API_KEY"
```

**Scale workers:**

```bash
curl --http1.1 -X POST https://api.yottalabs.ai/v2/serverless/12345/workers \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"workers": 2}'
```

## Common Issues and Fixes

<details>

<summary><code>python: command not found</code> in worker logs</summary>

The `vllm/vllm-openai` image uses `python3`, not `python`. Use `vllm serve ...` directly as the initialization command instead of `python -m vllm...`.

</details>

<details>

<summary><code>403 Forbidden</code> when downloading model weights</summary>

The model is gated and requires HuggingFace access approval. Either:

* Visit the model page on HuggingFace and request access, then add `HF_TOKEN` to `envVars`
* Switch to an open model like `Qwen/Qwen2.5-7B-Instruct` which requires no token

</details>

<details>

<summary><code>gpu type [X] not supported</code></summary>

Use the exact GPU type codes from the YottaLabs API. Common ones:

| Display Name   | API Code              |
| -------------- | --------------------- |
| RTX 4090 24 GB | `NVIDIA_RTX_4090_24G` |
| RTX 5090 32 GB | `NVIDIA_RTX_5090_32G` |
| A100 80 GB     | `NVIDIA_A100_80G`     |
| H100 80 GB     | `NVIDIA_H100_80G`     |

</details>

<details>

<summary><code>HTTP/2 stream error</code> with curl</summary>

Add `--http1.1` to your curl command to force HTTP/1.1.

</details>

## GPU and Model Selection Guide

| Model                         | VRAM Required | Recommended GPU    |
| ----------------------------- | ------------- | ------------------ |
| Qwen2.5-7B-Instruct (bf16)    | \~16 GB       | RTX 4090, RTX 5090 |
| Llama-3.1-8B-Instruct (bf16)  | \~16 GB       | RTX 4090, RTX 5090 |
| Qwen2.5-14B-Instruct (bf16)   | \~30 GB       | RTX 5090, A100 40G |
| Llama-3.1-70B-Instruct (bf16) | \~140 GB      | 2× A100 80G, H100  |

For models larger than your single GPU's VRAM, set `gpuCount` to 2 or more and add `--tensor-parallel-size 2` to the vLLM initialization command.

## Next Steps

* **Streaming responses**: Add `"stream": true` to the `input` payload and use a webhook to handle chunked responses
* **Quantized models**: Use AWQ or GPTQ variants (e.g. `Qwen/Qwen2.5-7B-Instruct-AWQ`) to reduce VRAM usage and increase throughput
* **Larger models**: Set `gpuCount: 2` and `--tensor-parallel-size 2` to run 14B or 70B models across multiple GPUs
* **Custom models**: Set `imageRegistry` and `credentialId` to pull from a private registry with your own fine-tuned model
