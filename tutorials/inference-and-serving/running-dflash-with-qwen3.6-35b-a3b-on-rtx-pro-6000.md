---
icon: d
---

# Running DFlash with Qwen3.6-35B-A3B with RTX PRO 6000

**DFlash** is a speculative decoding framework from [Z Lab](https://z-lab.ai/projects/dflash/) that uses a lightweight block diffusion model to draft multiple tokens in parallel, delivering **up to 6× lossless inference acceleration** over standard autoregressive decoding — and up to 2.5× faster than EAGLE-3. This guide walks you through deploying `Qwen3.6-35B-A3B` with DFlash on a Yotta Labs GPU Pod, using the `pod-templates-jupyterlab` template.

> **Note:** Yotta Labs provided compute resources for training the `Qwen3.6-35B-A3B-DFlash` draft model. The draft model is still under active training (2000 steps); early benchmark numbers are included at the end of this guide.

***

### Prerequisites

* A [Yotta Labs](https://www.yottalabs.ai/) account with sufficient credits
* A [Hugging Face](https://huggingface.co/) account — you must accept the model access conditions for [`z-lab/Qwen3.6-35B-A3B-DFlash`](https://huggingface.co/z-lab/Qwen3.6-35B-A3B-DFlash) and [`Qwen/Qwen3.6-35B-A3B`](https://huggingface.co/Qwen/Qwen3.6-35B-A3B) before downloading
* A Hugging Face access token (`HF_TOKEN`) with read permissions

#### GPU Requirements

`Qwen3.6-35B-A3B` is a Mixture-of-Experts model with 35B total parameters and \~3B active parameters per token. The BF16 weights alone are **71 GB**, so you need a GPU with sufficient VRAM to hold the full model plus the DFlash drafter and KV cache.

| GPU              | VRAM        | Strategy                   | Notes                                                                              |
| ---------------- | ----------- | -------------------------- | ---------------------------------------------------------------------------------- |
| **RTX PRO 6000** | **96 GB**   | single GPU                 | ✅ **Recommended on Yotta Labs** — fits BF16 comfortably with headroom for KV cache |
| 2× RTX 5090      | 64 GB total | `--tensor-parallel-size 2` | Good consumer-GPU alternative                                                      |
| H100 80GB        | 80 GB       | single GPU                 | Best for high-concurrency serving                                                  |
| A100 80GB        | 80 GB       | single GPU                 | Solid alternative; no native FP8 Tensor Cores                                      |

> **Why RTX PRO 6000?** With 96 GB VRAM, the full BF16 model (71 GB) + DFlash drafter (\~2 GB) + KV cache and runtime overhead all fit on a single card with comfortable headroom. No tensor parallelism needed, which simplifies deployment and avoids inter-GPU communication overhead.

***

### Step 1 — Deploy a Dflash Pod

* Log in to the [Yotta Labs Console](https://console.yottalabs.ai/).
* Navigate to **Compute → Pods** and click **Deploy** in the top right.
* On the **GPU Selection** page, choose **RTX PRO 6000/H200/2\*RTX5090**.
* Under **Pod Template**, select Dflash.

<figure><img src="../../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

*   Configure the following settings:

    | Setting       | Recommended Value                                                                |
    | ------------- | -------------------------------------------------------------------------------- |
    | System Volume | **200 GB** (BF16 weights ≈ 73 GB total for target + drafter, plus working space) |
    | GPU Count     | **1**                                                                            |
*   Add the following **Environment Variable** so your Hugging Face token is available inside the Pod:

    ```
    HF_TOKEN=<your_huggingface_token>
    ```
* Click **Deploy**. The Pod will enter `Initializing` state while resources are allocated.

> **Tip:** The System Volume must be at least 3× the image size. With BF16 weights for both models (\~73 GB) plus the JupyterLab base image, 200 GB gives you comfortable headroom for checkpoints and logs.

***

### Step 2 — Open JupyterLab

Once the Pod status shows **Running**:

1. Click **Connect** on the Pod card.
2. Under HTTP services, click the link for **port 8888** to open JupyterLab in your browser.
3. Create a new notebook: **File → New → Notebook**, select the Python 3 kernel.

All steps below run inside notebook cells. Commands prefixed with `%pip` or `!` run as shell commands inside the notebook.

***

### Step 3 — Install Dependencies

Run each of the following in separate notebook cells. Use `%pip` (not `pip`) so packages are installed into the active kernel.

vLLM has stable DFlash support built in. Use the nightly build until DFlash lands in a stable release.

```python
# Cell 1a — install vLLM nightly
%pip install -q -U vllm --torch-backend=auto --extra-index-url https://wheels.vllm.ai/nightly
%pip install -q huggingface_hub openai
```

> **After installing either option:** go to **Kernel → Restart Kernel** so the newly installed packages are picked up before running the next cells.

***

### Step 4 — Download Model Weights

Run this cell after restarting the kernel. Downloads go to `/root/models/`, which is persisted across Pod restarts.

```python
# Cell 2 — download models
import os
import subprocess

hf_token = os.environ.get("HF_TOKEN", "")
if not hf_token:
    raise ValueError(
        "HF_TOKEN is not set. Add it as a Pod environment variable in the "
        "Yotta Labs Console and redeploy, or set it manually below."
    )

# Download the target model (~71 GB, may take 15-25 min)
print("Downloading Qwen3.6-35B-A3B target model...")
subprocess.run([
    "huggingface-cli", "download", "Qwen/Qwen3.6-35B-A3B",
    "--local-dir", "/root/models/Qwen3.6-35B-A3B",
    "--token", hf_token
], check=True)

# Download the DFlash draft model (~2 GB)
# Requires accepting access conditions at:
# https://huggingface.co/z-lab/Qwen3.6-35B-A3B-DFlash
print("Downloading DFlash draft model...")
subprocess.run([
    "huggingface-cli", "download", "z-lab/Qwen3.6-35B-A3B-DFlash",
    "--local-dir", "/root/models/Qwen3.6-35B-A3B-DFlash",
    "--token", hf_token
], check=True)

print("✅ All downloads complete.")
```

> **Note:** Total download is approximately 73 GB. This may take 15–30 minutes. Because `/root` is persisted on Yotta Labs Pods, the weights survive Pod restarts and kernel restarts — you only need to download once.

***

### Step 5 — Send Your First Request

Run this in a new cell after the server shows ✅ ready:

```python
# Cell 4 — test inference
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:30000/v1",
    api_key="EMPTY"
)

response = client.chat.completions.create(
    model="Qwen/Qwen3.6-35B-A3B",
    messages=[{"role": "user", "content": "Give a brief introduction on Hermes Trismegistus"}],
    max_tokens=4096,
    temperature=0.0
)

print(response.choices[0].message.content)
```

Optionally run a health check before your first request:

```python
# Cell 4a — optional health check
import urllib.request
try:
    with urllib.request.urlopen("http://localhost:30000/health") as r:
        print("Server status:", r.status, r.read().decode())
except Exception as e:
    print("Server not ready yet:", e)
```

***

### Performance Reference

The following acceptance lengths are measured with the `Qwen3.6-35B-A3B-DFlash` draft model (thinking enabled, block size 16, SGLang backend). Higher acceptance length = more tokens accepted per draft step = faster effective throughput.

| Dataset   | Accept Length |
| --------- | ------------- |
| GSM8K     | 5.8           |
| Math500   | 6.3           |
| HumanEval | 5.2           |
| MBPP      | 4.8           |
| MT-Bench  | 4.4           |

> These are early results — the draft model is still under training. Numbers will improve as training progresses.

***

### Tips & Troubleshooting

**Model download fails / access denied** Accept the model access conditions at [huggingface.co/z-lab/Qwen3.6-35B-A3B-DFlash](https://huggingface.co/z-lab/Qwen3.6-35B-A3B-DFlash) while logged into Hugging Face, then re-run Cell 2.

**`HF_TOKEN` is empty inside the notebook** The token must be set as a Pod environment variable before deployment. If you forgot, terminate and redeploy with `HF_TOKEN` added. As a temporary workaround you can set it directly in a cell:

```python
import os
os.environ["HF_TOKEN"] = "hf_your_token_here"
```

**Packages not found after install** Always run **Kernel → Restart Kernel** after running Cell 1. `%pip install` takes effect only after the kernel is restarted.

**Server process exits immediately** Check the log for errors:

```python
!tail -80 /root/vllm_server.log
# or for SGLang:
!tail -80 /root/sglang_server.log
```

**Out of GPU memory (OOM)** Change `"0.92"` to `"0.88"` in the `--gpu-memory-utilization` argument in Cell 3, then re-run. If the problem persists, add `"--max-model-len", "65536"` to cap the context window and free KV cache budget.

**Server not reachable on port 30000** Make sure port 30000 was configured as an HTTP port when deploying the Pod. Click **Connect** on the Pod card — port 30000 should appear in the HTTP services list. If not, you need to redeploy with the port added.

**Kernel restarted and server is gone** The server subprocess is tied to the kernel session. If the kernel restarts, re-run Cell 3. Model weights in `/root/models/` are still on disk — no re-download needed.

**Slow generation / DFlash not kicking in** DFlash is most effective at low-to-medium batch sizes (1–32 concurrent requests). For high concurrency, add `"--speculative-disable-by-batch-size", "32"` to the argument list in Cell 3a.

***

### Additional Resources

* [DFlash Paper (arXiv)](https://arxiv.org/abs/2602.06036)
* [DFlash GitHub](https://github.com/z-lab/dflash)
* [z-lab/Qwen3.6-35B-A3B-DFlash on Hugging Face](https://huggingface.co/z-lab/Qwen3.6-35B-A3B-DFlash)
* [Yotta Labs GPU Pods Documentation](https://docs.yottalabs.ai/products/gpu-pods)
* [Yotta Labs Quickstart](https://docs.yottalabs.ai/products/quickstart)
* [DFlash Feedback Form](https://forms.gle/4YNwfqb4nJdqn6hq9)
