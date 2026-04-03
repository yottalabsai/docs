# Serverless Inference with Image Generation models

This tutorial walks you through deploying **FLUX.1-Dev** — a high-quality text-to-image model by Black Forest Labs — as a production-ready serverless endpoint on YottaLabs. The deployment uses the official YottaLabs ComfyUI runtime image, requiring zero custom Docker builds.

***

### What You Will Build

A serverless FLUX.1-Dev endpoint on YottaLabs that:

* Runs **FLUX.1-Dev** on an NVIDIA RTX 5090 (32 GB VRAM)
* Serves a **ComfyUI HTTP API** on port 8188
* Accepts text prompts and returns **Base64-encoded images**
* Can be called from any HTTP client or Python script

***

### Prerequisites

* A YottaLabs account with an API key (`x-api-key`)
* A Hugging Face token (`HF_TOKEN`) with read access approved for:
  * `black-forest-labs/FLUX.1-dev`
  * `comfyanonymous/flux_text_encoders`
* `curl` and `jq` installed locally

> **Approving HuggingFace access:** Visit [https://huggingface.co/black-forest-labs/FLUX.1-dev](https://huggingface.co/black-forest-labs/FLUX.1-dev) and click **Request access**. Approval is typically granted within a few minutes.

***

### Architecture Overview

```
Your Client
    │
    │  POST /v1/chat/completions  (with prompt)
    ▼
YottaLabs Gateway  (Bearer token auth)
    │
    ▼
Worker (RTX 5090)
    │
    ├── /start.sh
    │     ├── Downloads FLUX.1-Dev weights from HuggingFace (~24 GB)
    │     └── Starts ComfyUI on port 8188
    │
    └── ComfyUI API
          ├── POST /prompt        → submit generation job
          ├── GET  /history/{id}  → poll for result
          └── GET  /view?...      → fetch image bytes → Base64
```

The official YottaLabs image handles everything inside the container automatically: model downloading, text encoder setup, and ComfyUI startup. You only need to submit prompts via the ComfyUI HTTP API.

***

### Step 1 — Create the Serverless Endpoint

```bash
curl --http1.1 -X POST https://api.yottalabs.ai/v2/serverless \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "flux-dev-comfyui",
    "image": "yottalabsai/flux1.dev:comfyui-cuda12.8.1-ubuntu22.04-2025102101",
    "serviceMode": "ALB",
    "workers": 1,
    "containerVolumeInGb": 100,
    "resources": [
      {
        "region": "us-east",
        "gpuType": "NVIDIA_RTX_5090_32G",
        "gpuCount": 1
      }
    ],
    "expose": {
      "port": 8188,
      "protocol": "http"
    },
    "envVars": [
      {"key": "HF_TOKEN", "value": "YOUR_HF_TOKEN"},
      {"key": "ENABLE_FLUX_VAE", "value": "true"},
      {"key": "FLUX_MODEL_DIR", "value": "/home/ubuntu/ComfyUI/models"}
    ],
    "initializationCommand": "sudo -E /start.sh"
  }'
```

**Key parameters explained:**

| Parameter               | Value                               | Notes                                                       |
| ----------------------- | ----------------------------------- | ----------------------------------------------------------- |
| `image`                 | `yottalabsai/flux1.dev:comfyui-...` | Official YottaLabs FLUX image with ComfyUI runtime          |
| `serviceMode`           | `ALB`                               | Direct proxy mode — requests go straight to ComfyUI         |
| `containerVolumeInGb`   | `100`                               | FLUX.1-Dev weights + text encoders are \~24 GB total        |
| `expose.port`           | `8188`                              | ComfyUI's default HTTP port                                 |
| `HF_TOKEN`              | your token                          | Required to download gated FLUX.1-Dev weights               |
| `ENABLE_FLUX_VAE`       | `true`                              | Downloads and loads the VAE (`ae.safetensors`)              |
| `initializationCommand` | `sudo -E /start.sh`                 | Runs the official startup script (model download + ComfyUI) |

Save the `id` and `domain` from the response — you will need them for all subsequent calls.

***

### Step 2 — Wait for the Endpoint to Be Ready

The first startup takes **10–20 minutes** because the container must download \~24 GB of model weights from HuggingFace. Subsequent starts are fast if weights are already cached on persistent storage.

**Check the endpoint status:**

```bash
curl --http1.1 https://api.yottalabs.ai/v2/serverless \
  -H "x-api-key: YOUR_API_KEY" | python3 -m json.tool
```

Wait for `"status": "RUNNING"`.

**Then verify ComfyUI is up by checking system stats:**

```bash
curl --http1.1 https://YOUR_ENDPOINT_DOMAIN/system_stats \
  -H "Authorization: Bearer YOUR_API_KEY"
```

A healthy response looks like:

```json
{
  "system": {
    "os": "posix",
    "python_version": "3.12.x",
    "embedded_python": false
  },
  "devices": [
    {
      "name": "NVIDIA GeForce RTX 5090",
      "type": "cuda",
      "vram_total": 34359738368,
      "vram_free": 28000000000
    }
  ]
}
```

**Check the model download log if startup is taking long:**

```bash
curl --http1.1 https://YOUR_ENDPOINT_DOMAIN/view?filename=flux_download.log \
  -H "Authorization: Bearer YOUR_API_KEY"
```

***

### Step 3 — Submit an Image Generation Request

ComfyUI uses a **workflow JSON** format to describe the generation pipeline. Below is a minimal FLUX.1-Dev workflow that takes a text prompt and generates a 1024×1024 image.

#### 3.1 — The Workflow Payload

Save this as `flux_workflow.json`:

```json
{
  "prompt": {
    "6": {
      "class_type": "CLIPTextEncode",
      "inputs": {
        "clip": ["11", 0],
        "text": "a cinematic portrait of an astronaut on the moon, golden hour, highly detailed"
      }
    },
    "8": {
      "class_type": "VAEDecode",
      "inputs": {
        "samples": ["13", 0],
        "vae": ["10", 0]
      }
    },
    "9": {
      "class_type": "SaveImage",
      "inputs": {
        "filename_prefix": "flux_output",
        "images": ["8", 0]
      }
    },
    "10": {
      "class_type": "VAELoader",
      "inputs": {
        "vae_name": "ae.safetensors"
      }
    },
    "11": {
      "class_type": "DualCLIPLoader",
      "inputs": {
        "clip_name1": "t5xxl_fp8_e4m3fn_scaled.safetensors",
        "clip_name2": "clip_l.safetensors",
        "type": "flux"
      }
    },
    "12": {
      "class_type": "UNETLoader",
      "inputs": {
        "unet_name": "flux1-dev.safetensors",
        "weight_dtype": "fp8_e4m3fn"
      }
    },
    "13": {
      "class_type": "KSampler",
      "inputs": {
        "cfg": 1.0,
        "denoise": 1.0,
        "latent_image": ["14", 0],
        "model": ["12", 0],
        "negative": ["15", 0],
        "positive": ["6", 0],
        "sampler_name": "euler",
        "scheduler": "simple",
        "seed": 42,
        "steps": 20
      }
    },
    "14": {
      "class_type": "EmptyLatentImage",
      "inputs": {
        "batch_size": 1,
        "height": 1024,
        "width": 1024
      }
    },
    "15": {
      "class_type": "CLIPTextEncode",
      "inputs": {
        "clip": ["11", 0],
        "text": ""
      }
    }
  }
}
```

#### 3.2 — Submit the Prompt

```bash
curl --http1.1 -X POST https://YOUR_ENDPOINT_DOMAIN/prompt \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d @flux_workflow.json
```

**Response:**

```json
{
  "prompt_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "number": 1,
  "node_errors": {}
}
```

Save the `prompt_id` — you will poll with it in the next step.

***

### Step 4 — Poll for the Result

```bash
curl --http1.1 https://YOUR_ENDPOINT_DOMAIN/history/a1b2c3d4-e5f6-7890-abcd-ef1234567890 \
  -H "Authorization: Bearer YOUR_API_KEY" | python3 -m json.tool
```

When generation is complete, the response contains the output image filename:

```json
{
  "a1b2c3d4-e5f6-7890-abcd-ef1234567890": {
    "status": {
      "status_str": "success",
      "completed": true
    },
    "outputs": {
      "9": {
        "images": [
          {
            "filename": "flux_output_00001_.png",
            "subfolder": "",
            "type": "output"
          }
        ]
      }
    }
  }
}
```

***

### Step 5 — Fetch the Image as Base64

Use the filename from the history response to download and encode the image:

```bash
curl --http1.1 \
  "https://YOUR_ENDPOINT_DOMAIN/view?filename=flux_output_00001_.png&type=output" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --output generated.png
```

**To get Base64 directly:**

```bash
curl --http1.1 \
  "https://YOUR_ENDPOINT_DOMAIN/view?filename=flux_output_00001_.png&type=output" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --output - | base64
```

***

### Step 6 — Full Python Client

This script wraps the entire flow — submit prompt, poll until done, return Base64:

```python
import requests
import base64
import time
import json

ENDPOINT = "https://YOUR_ENDPOINT_DOMAIN"
API_KEY = "YOUR_API_KEY"
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}

def build_workflow(prompt: str, width: int = 1024, height: int = 1024, steps: int = 20, seed: int = 42) -> dict:
    return {
        "prompt": {
            "6":  {"class_type": "CLIPTextEncode",   "inputs": {"clip": ["11", 0], "text": prompt}},
            "8":  {"class_type": "VAEDecode",         "inputs": {"samples": ["13", 0], "vae": ["10", 0]}},
            "9":  {"class_type": "SaveImage",         "inputs": {"filename_prefix": "flux_output", "images": ["8", 0]}},
            "10": {"class_type": "VAELoader",         "inputs": {"vae_name": "ae.safetensors"}},
            "11": {"class_type": "DualCLIPLoader",    "inputs": {"clip_name1": "t5xxl_fp8_e4m3fn_scaled.safetensors", "clip_name2": "clip_l.safetensors", "type": "flux"}},
            "12": {"class_type": "UNETLoader",        "inputs": {"unet_name": "flux1-dev.safetensors", "weight_dtype": "fp8_e4m3fn"}},
            "13": {"class_type": "KSampler",          "inputs": {"cfg": 1.0, "denoise": 1.0, "latent_image": ["14", 0], "model": ["12", 0], "negative": ["15", 0], "positive": ["6", 0], "sampler_name": "euler", "scheduler": "simple", "seed": seed, "steps": steps}},
            "14": {"class_type": "EmptyLatentImage",  "inputs": {"batch_size": 1, "height": height, "width": width}},
            "15": {"class_type": "CLIPTextEncode",    "inputs": {"clip": ["11", 0], "text": ""}},
        }
    }

def generate_image(prompt: str, width: int = 1024, height: int = 1024, steps: int = 20, seed: int = 42, timeout: int = 300) -> str:
    """
    Submit a prompt to FLUX.1-Dev and return the result as a Base64-encoded PNG string.
    """
    # 1. Submit prompt
    workflow = build_workflow(prompt, width, height, steps, seed)
    resp = requests.post(f"{ENDPOINT}/prompt", headers=HEADERS, json=workflow)
    resp.raise_for_status()
    prompt_id = resp.json()["prompt_id"]
    print(f"Submitted. prompt_id: {prompt_id}")

    # 2. Poll for completion
    start = time.time()
    while True:
        if time.time() - start > timeout:
            raise TimeoutError(f"Generation timed out after {timeout}s")
        
        history = requests.get(f"{ENDPOINT}/history/{prompt_id}", headers=HEADERS).json()
        
        if prompt_id in history:
            job = history[prompt_id]
            if job["status"]["completed"]:
                images = job["outputs"]["9"]["images"]
                filename = images[0]["filename"]
                subfolder = images[0]["subfolder"]
                print(f"Done. Fetching: {filename}")
                break
        
        print("Waiting for generation...")
        time.sleep(3)

    # 3. Fetch image and encode as Base64
    params = {"filename": filename, "type": "output"}
    if subfolder:
        params["subfolder"] = subfolder
    
    img_resp = requests.get(f"{ENDPOINT}/view", headers=HEADERS, params=params)
    img_resp.raise_for_status()
    
    return base64.b64encode(img_resp.content).decode("utf-8")


if __name__ == "__main__":
    prompt = "a cinematic portrait of an astronaut on the moon, golden hour, highly detailed, 8k"
    
    b64 = generate_image(prompt, width=1024, height=1024, steps=20, seed=42)
    
    # Save to file
    with open("output.png", "wb") as f:
        f.write(base64.b64decode(b64))
    
    print(f"Image saved to output.png")
    print(f"Base64 length: {len(b64)} characters")
```

***

### Managing Your Endpoint

**Stop the endpoint (pause billing):**

```bash
curl --http1.1 -X POST https://api.yottalabs.ai/v2/serverless/{id}/stop \
  -H "x-api-key: YOUR_API_KEY"
```

**Restart:**

```bash
curl --http1.1 -X POST https://api.yottalabs.ai/v2/serverless/{id}/start \
  -H "x-api-key: YOUR_API_KEY"
```

**Check the generation queue:**

```bash
curl --http1.1 https://YOUR_ENDPOINT_DOMAIN/queue \
  -H "Authorization: Bearer YOUR_API_KEY" | python3 -m json.tool
```

***

### Common Issues and Fixes

**Model weights not downloaded yet (generation fails immediately)**

Check the download log and wait:

```bash
curl --http1.1 "https://YOUR_ENDPOINT_DOMAIN/view?filename=flux_download.log" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

The full download is \~24 GB and takes 10–20 minutes on first start.

**403 on HuggingFace during download**

Your `HF_TOKEN` does not have approved access to `black-forest-labs/FLUX.1-dev`. Visit the model page and request access, then recreate the endpoint with the correct token.

**`node_errors` in the prompt response**

A model file is missing or named differently. Check that all files exist under `/home/ubuntu/ComfyUI/models/`:

```
diffusion_models/flux1-dev.safetensors
vae/ae.safetensors
text_encoders/t5xxl_fp8_e4m3fn_scaled.safetensors
text_encoders/clip_l.safetensors
```

**Generation is slow (>60s per image)**

Normal for FLUX.1-Dev at 20 steps on FP16. Options to speed up:

* Reduce `steps` to 10–15 (quality tradeoff)
* Use the Nunchaku-optimized image instead: `yottalabsai/flux1.dev:comfyui-nunchaku-cuda12.8.1-ubuntu22.04-2025102101`

***

### Performance Reference

| Steps | Resolution           | RTX 5090 (approx.) |
| ----- | -------------------- | ------------------ |
| 10    | 512×512              | \~8s               |
| 20    | 1024×1024            | \~25s              |
| 20    | 1024×1024 (Nunchaku) | \~12s              |

***

### Nunchaku Variant (Faster Inference)

YottaLabs also provides a quantized version of the image with Nunchaku acceleration, which roughly halves generation time on RTX 5090:

```bash
"image": "yottalabsai/flux1.dev:comfyui-nunchaku-cuda12.8.1-ubuntu22.04-2025102101"
```

Everything else — environment variables, ports, API calls — remains identical. Switch the image name and redeploy.

***

### Next Steps

* **LoRA support**: Place `.safetensors` LoRA files under `ComfyUI/models/loras/` and add a `LoraLoader` node to the workflow
* **Batch generation**: Set `batch_size` > 1 in the `EmptyLatentImage` node to generate multiple images per request
* **Different resolutions**: FLUX.1-Dev supports arbitrary resolutions. Common choices: 768×1344 (portrait), 1344×768 (landscape), 1024×1024 (square)
* **img2img**: Add an `ImageToLatent` node before the KSampler and set `denoise` < 1.0
