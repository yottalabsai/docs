---
icon: whale
---

# Run DeepSeek V4 Flash/Pro on B300

Target hardware: **8× B300 (192GB each, 1.5TB total VRAM)**\
Covers **V4-Flash** (284B / 13B active, \~146GB) and **V4-Pro** (1.6T / 49B active, \~960GB)\
Both models are MIT licensed.

Created based on: [https://www.lmsys.org/blog/2026-04-25-deepseek-v4/](https://www.lmsys.org/blog/2026-04-25-deepseek-v4/)

***

## 1. Spin Up the Virtual Machine

Log in to [yottalabs.ai](https://www.yottalabs.ai/) → **Compute → VMs→ Launch**.

For those who intend to run V4-Flash, two H200 SXM in a single pod is enough. If you need full 1M context or high QPS, choose 8× H200, which can cost significantly more.

For V4-Pro, 8× HGX B300 is the its native FP4 execution, full 1M context, no model-len cap. H200 cluster nodes is also a good choice, if there are not as many  B300 as we want.

### Pod Settings

| Field             | Value                                   |
| ----------------- | --------------------------------------- |
| **GPU Type**      | B300                                    |
| **GPU Count**     | 8                                       |
| **Image**         | `lmsysorg/sglang:deepseek-v4-blackwell` |
| **System Volume** | ≥ 300 GB (Flash) / ≥ 1.2 TB (Pro)       |

{% hint style="warning" %}
**System volume sizing rule:** The SGLang Blackwell image is \~15GB. Yotta Labs requires volume ≥ image size × 3, plus model weights plus buffer:

* Flash: 45GB + 200GB + 55GB buffer = **≥ 300GB**
* Pro: 45GB + 1000GB + 155GB buffer = **≥ 1.2TB**


{% endhint %}

Once deployed, click **Connect → SSH** to open a terminal.

***

## 2. Environment Setup

### Set HuggingFace token

```bash
export HF_TOKEN="hf_your_token_here"
echo 'export HF_TOKEN="hf_your_token_here"' >> ~/.bashrc
```

> Get a token at [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens).

### Install the `hf` CLI

Ubuntu 24.04 has an externally managed Python. Install to user directory:

```bash
pip install -U "huggingface_hub[cli]" hf_transfer --user --quiet

# Add to PATH
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc

# Enable fast parallel downloads
export HF_HUB_ENABLE_HF_TRANSFER=1
echo 'export HF_HUB_ENABLE_HF_TRANSFER=1' >> ~/.bashrc

# Verify
hf version
```

> ⚠️ Do NOT use `huggingface-cli` — it may not be on PATH. Use `hf` instead.

### Accept model licenses on HuggingFace

Visit each link and click **"Access repository"**:

* https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash ← for Flash
* https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro ← for Pro

### Verify GPUs

```bash
nvidia-smi
# Expected: 8× B300, driver ≥ 570, CUDA ≥ 12.8
```

***

## 3. Download Model Weights

This could take 40\~60 minutes depends on which version you choose.

{% hint style="warning" %}
Always download to `/home/ubuntu/models/` — the pod runs as the `ubuntu` user and **cannot write to `/root/`**.
{% endhint %}

```bash
mkdir -p /home/ubuntu/models
```

> **Run downloads inside tmux** — SSH disconnects will kill the process otherwise.\
> `hf download` supports resume: if interrupted, just re-run the same command and it will skip already-completed files. If you see `Still waiting to acquire lock`, a previous process is still running — find and kill it first with `pkill -f "hf download"`, then delete lock files with `find /home/ubuntu/models -name "*.lock" -delete`.

{% tabs %}
{% tab title="V4-Flash" %}
#### Option A — V4-Flash (\~146GB, FP4+FP8)

```bash
tmux new-session -d -s download \
  "hf download deepseek-ai/DeepSeek-V4-Flash \
   --local-dir /home/ubuntu/models/deepseek-v4-flash \
   --token $HF_TOKEN"

tmux attach -t download
# ~10–20 min on datacenter uplink
```
{% endtab %}

{% tab title="V4-Pro" %}
#### Option B — V4-Pro (\~960GB, FP4+FP8)

```bash
tmux new-session -d -s download \
  "hf download deepseek-ai/DeepSeek-V4-Pro \
   --local-dir /home/ubuntu/models/deepseek-v4-pro \
   --token $HF_TOKEN"

tmux attach -t download
# 2–4 hours — do not close the terminal
```

Monitor progress in a second window:

```bash
# Open a new SSH session or tmux pane, then:
watch -n 10 "du -sh /home/ubuntu/models/deepseek-v4-pro"
```

Verify completion:

```bash
ls /home/ubuntu/models/deepseek-v4-pro | wc -l   # expect: 92 files
ls /home/ubuntu/models/deepseek-v4-pro/config.json  # must exist
```
{% endtab %}
{% endtabs %}

***

## 4. Launch SGLang

Decide which image to use based on what GPUs you chose:

<figure><img src="../../.gitbook/assets/image (33).png" alt="" width="563"><figcaption></figcaption></figure>

> Chart from [https://docs.sglang.io/cookbook/autoregressive/DeepSeek/DeepSeek-V4](https://docs.sglang.io/cookbook/autoregressive/DeepSeek/DeepSeek-V4)

**Key points for B300 (Blackwell):**

* Use image `lmsysorg/sglang:deepseek-v4-b300`
* Use official `deepseek-ai/` checkpoints (native FP4+FP8)
* Enable `--moe-runner-backend flashinfer_mxfp4` for FP4 expert kernel
* Mount model as `-v /home/ubuntu/models/<model>:/workspace/model` and pass `--model-path /workspace/model`
* Always use `sudo docker` in our virtual machine

{% tabs %}
{% tab title="V4-Flash" %}
#### 4a. V4-Flash on 8× B300

```bash
sudo docker run -d --gpus all \
  --shm-size 32g \
  --ipc=host \
  -p 8000:8000 \
  -e HF_TOKEN=$HF_TOKEN \
  -e PYTORCH_ALLOC_CONF=expandable_segments:True \
  -v /home/ubuntu/models/deepseek-v4-flash:/workspace/model \
  lmsysorg/sglang:deepseek-v4-blackwell \
  python3 -m sglang.launch_server \
    --model-path /workspace/model \
    --served-model-name deepseek-v4-flash \
    --tp 8 \
    --host 0.0.0.0 \
    --port 8000 \
    --trust-remote-code \
    --mem-fraction-static 0.88 \
    --context-length 524288 \
    --moe-runner-backend flashinfer_mxfp4 \
    --speculative-algo EAGLE \
    --speculative-num-steps 3 \
    --speculative-eagle-topk 1 \
    --speculative-num-draft-tokens 4 \
    --chunked-prefill-size 4096 \
    --disable-flashinfer-autotune
```
{% endtab %}

{% tab title="V4-Pro" %}
#### 4b. V4-Pro on 8× B200 ← Full "满血" model

```bash
sudo docker run -d --gpus all \
  --shm-size 64g \
  --ipc=host \
  -p 8000:8000 \
  -e HF_TOKEN=$HF_TOKEN \
  -e PYTORCH_ALLOC_CONF=expandable_segments:True \
  -v /home/ubuntu/models/deepseek-v4-pro:/workspace/model \
  lmsysorg/sglang:deepseek-v4-blackwell \
  python3 -m sglang.launch_server \
    --model-path /workspace/model \
    --served-model-name deepseek-v4-pro \
    --tp 8 \
    --host 0.0.0.0 \
    --port 8000 \
    --trust-remote-code \
    --mem-fraction-static 0.85 \
    --context-length 131072 \
    --moe-runner-backend flashinfer_mxfp4 \
    --speculative-algo EAGLE \
    --speculative-num-steps 3 \
    --speculative-eagle-topk 1 \
    --speculative-num-draft-tokens 4 \
    --chunked-prefill-size 4096 \
    --disable-flashinfer-autotune
```

> Start with `--context-length` 131072 (128K). Once confirmed working, increase to `262144` or higher if `nvidia-smi` shows headroom.
{% endtab %}
{% endtabs %}

### Watch startup logs

```bash
sudo docker logs -f $(sudo docker ps -q)
# Startup takes 5–10 min (model loading + JIT kernel compilation)
# Ready when you see: "The server is fired up and ready to roll!"
```

### Key Flag Reference

| Flag                                          | Purpose                                                 |
| --------------------------------------------- | ------------------------------------------------------- |
| `--tp 8`                                      | Tensor parallelism across all 8 B300s                   |
| `--moe-runner-backend flashinfer_mxfp4`       | **Required on B300** — enables FP4 MoE kernel           |
| `--mem-fraction-static`                       | VRAM fraction for weights; leave 0.12–0.15 for KV cache |
| `--context-length`                            | Start at 131072, increase if VRAM allows                |
| `--speculative-algo EAGLE`                    | Speculative decoding, \~2.5× token accept rate          |
| `--shm-size 64g`                              | Shared memory for inter-GPU comms (use 64g for Pro)     |
| `PYTORCH_ALLOC_CONF=expandable_segments:True` | Reduces memory fragmentation                            |

***

## 5. Verify the Server

### Health check

```bash
curl http://<pod-public-ip>:8000/v1/models
# {"object":"list","data":[{"id":"deepseek-v4-pro",...}]}
```

### Basic inference

```bash
curl http://<pod-public-ip>:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-v4-pro",
    "messages": [{"role": "user", "content": "Hello, what model are you?"}],
    "max_tokens": 256,
    "temperature": 1.0,
    "top_p": 1.0
  }'
```

> **Always use `temperature=1.0, top_p=1.0`** — DeepSeek officially recommends these for V4. Lowering temperature degrades reasoning quality on MoE architectures.

### Python client

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="EMPTY"
)

response = client.chat.completions.create(
    model="deepseek-v4-pro",
    messages=[{"role": "user", "content": "Write a Python function to merge two sorted lists."}],
    max_tokens=1024,
    temperature=1.0,
    top_p=1.0
)
print(response.choices[0].message.content)
```

***

## 6. Enabling Thinking Mode

V4 has three reasoning levels: **Non-think**, **Think High**, **Think Max**.

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

# Think High (recommended for most tasks)
response = client.chat.completions.create(
    model="deepseek-v4-pro",
    messages=[{"role": "user", "content": "Prove that sqrt(2) is irrational."}],
    max_tokens=8192,
    temperature=1.0,
    top_p=1.0,
    extra_body={"chat_template_kwargs": {"thinking": True}}
)

if hasattr(response.choices[0].message, "reasoning_content"):
    print("=== Thinking ===")
    print(response.choices[0].message.reasoning_content)
print("=== Answer ===")
print(response.choices[0].message.content)
```

> **Think Max** requires `--context-length` ≥ 384K. Achievable on 8× B200 with Flash; for Pro use Think High (128K) until you've confirmed stable operation.

### Streaming with thinking

```python
response = client.chat.completions.create(
    model="deepseek-v4-pro",
    messages=[{"role": "user", "content": "What is 15% of 240?"}],
    max_tokens=2048,
    extra_body={"chat_template_kwargs": {"thinking": True}},
    stream=True,
)

thinking_started = False
for chunk in response:
    if not chunk.choices:
        continue
    delta = chunk.choices[0].delta
    if getattr(delta, "reasoning_content", None):
        if not thinking_started:
            print("=== Thinking ===", flush=True)
            thinking_started = True
        print(delta.reasoning_content, end="", flush=True)
    if delta.content:
        if thinking_started:
            print("\n=== Answer ===", flush=True)
            thinking_started = False
        print(delta.content, end="", flush=True)
```

***

## Summary Cheat Sheet

```
Hardware:   8× B200 (Blackwell — native FP4+FP8)
Image:      lmsysorg/sglang:deepseek-v4-blackwell
Docker:     always  sudo docker  (not docker)
Models:     /home/ubuntu/models/   (not /root/)
Mount:      -v /home/ubuntu/models/<model>:/workspace/model
Path flag:  --model-path /workspace/model

Flash:   deepseek-ai/DeepSeek-V4-Flash   ~146GB   --tp 8   512K+ context
Pro:     deepseek-ai/DeepSeek-V4-Pro     ~960GB   --tp 8   128K→1M context

MoE flag:   --moe-runner-backend flashinfer_mxfp4   ← required on B200
Sampling:   temperature=1.0  top_p=1.0              ← do not change

Download:   hf download deepseek-ai/<model> --local-dir /home/ubuntu/models/<model> --token $HF_TOKEN
Resume:     pkill -f "hf download" && find /home/ubuntu/models -name "*.lock" -delete && re-run
```

***

_References:_ [_SGLang DeepSeek-V4 Docs_](https://docs.sglang.io/cookbook/autoregressive/DeepSeek/DeepSeek-V4) _·_ [_LMSYS Day-0 Blog_](https://www.lmsys.org/blog/2026-04-25-deepseek-v4/) _·_ [_Yotta Labs Pod Docs_](https://docs.yottalabs.ai/products/gpu-pods)
