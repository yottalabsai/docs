---
icon: aws
---

# LLM Serving on Yotta Labs with AWS Trainium

> This tutorial walks you through deploying an LLM inference service on the Yotta Labs platform using an **AWS Trainium** Pod.\
> Reference: [AWS Neuron Docs — vLLM on Neuron](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/libraries/nxd-inference/vllm/index.html) (Neuron SDK 2.29.0)

## 1. Platform Overview & Hardware Selection

Log in to [https://yottalabs.ai](https://yottalabs.ai/), go to the **Pods** page, and click **Deploy**.

In the hardware selector, switch to the **AWS** tab (NVIDIA is the default):

```
NVIDIA  |  [AWS]   ← click to switch
```

Select **Trainium1**:

| Spec  | Value      |
| ----- | ---------- |
| VRAM  | 32 GB      |
| RAM   | 32 GB      |
| vCPU  | 8          |
| Price | $1.40 / Hr |

<figure><img src="../.gitbook/assets/image (217).png" alt=""><figcaption></figcaption></figure>

## 2. AWS Trainium vs NVIDIA GPU — Key Differences

| Aspect                   | NVIDIA GPU                        | AWS Trainium1                                                           |
| ------------------------ | --------------------------------- | ----------------------------------------------------------------------- |
| **Compute backend**      | CUDA / cuDNN                      | AWS Neuron SDK (`neuronx-distributed-inference`)                        |
| **Compilation**          | JIT / eager                       | **AOT compilation** — model is compiled to `.neff` format on first run  |
| **First startup time**   | 1–3 min (download weights only)   | **\~20 min** (includes Neuron AOT compilation)                          |
| **Subsequent startup**   | Same as above                     | Seconds–1 min (compiled artifacts cached)                               |
| **vLLM integration**     | Native (`pip install vllm`)       | `vllm-neuron` plugin — **pre-installed** in the Yotta Neuron image      |
| **Backend flag**         | None (cuda by default)            | `VLLM_NEURON_FRAMEWORK=neuronx-distributed-inference`                   |
| **Parallelism**          | `--tensor-parallel-size` = # GPUs | `--tensor-parallel-size` = # NeuronCores                                |
| **Memory mgmt**          | `torch.cuda.*`                    | `torch_xla` + Neuron Runtime                                            |
| **Monitoring**           | `nvidia-smi`, `nvtop`             | `neuron-top`, `neuron-ls`, `neuron-monitor`                             |
| **Compiled model cache** | Not needed                        | Set `NEURON_COMPILED_ARTIFACTS` to persist across restarts              |
| **Weight sharding**      | Re-sharded every run              | Can be persisted with `save_sharded_checkpoint: true`                   |
| **CUDA libs**            | Required                          | Not present — `libcuda.so` warnings on import are expected and harmless |

### Key Takeaways

* The `vllm-neuron` plugin exposes the **same OpenAI-compatible `/v1` API** as standard vLLM — no client-side changes needed.
* The Neuron plugin is **pre-installed** in the `yottalabsai/pytorch-inference-vlm-neuronx` image. No manual installation required.
* The Triton warning (`0 active driver(s) found`) and `libcuda.so` import warning are **expected** on Trainium and do not affect inference.

## 3. Launching a Trainium Pod

### 3.1 Select the Image

| Field            | Value                                                                                  |
| ---------------- | -------------------------------------------------------------------------------------- |
| **Image Source** | Docker Hub                                                                             |
| **Image Type**   | Public Image                                                                           |
| **Image**        | `yottalabsai/pytorch-inference-vlm-neuronx:0.13.0-neuronx-py312-sdk2.27.1-ubuntu24.04` |

This image includes:

* `torch-neuronx 2.9.0` + `neuronxcc` (Neuron compiler)
* `neuronx-distributed` (NxD Inference library)
* `vllm 0.13.0` with the `vllm-neuron` plugin pre-installed and auto-activated
* `transformers 4.56.2`, `torch_xla 2.9.0`

<figure><img src="../.gitbook/assets/image (218).png" alt=""><figcaption></figcaption></figure>

### 3.2 Ports

Expose these ports in addition to the defaults (22, 8080):

| Port | Service                    |
| ---- | -------------------------- |
| 8000 | vLLM OpenAI-compatible API |
| 8888 | JupyterLab                 |

### 3.3 Deploy

Review the pricing summary (\~$1.40/Hr + storage) and click **Deploy**. The Pod reaches **Running** state in 1–2 minutes (image pull).

## 4. Connecting & Verifying the Environment

### 4.1 SSH into the Pod

```bash
ssh root@<pod-ip> -p 22
```

### 4.2 Activate the Conda Environment

{% hint style="warning" %}
**Critical**: All Neuron packages live in the conda base environment at `/opt/conda`. The system Python at `/usr/bin/python3` has no packages installed. You must activate conda first.
{% endhint %}

```bash
source /opt/conda/bin/activate

# Verify the Python path has switched
which python3
# Expected: /opt/conda/bin/python3
```

To make this permanent across SSH sessions:

```bash
echo 'source /opt/conda/bin/activate' >> ~/.bashrc
source ~/.bashrc
```

### 4.3 Verify Neuron Devices

```bash
# List Neuron devices (equivalent to nvidia-smi -L)
neuron-ls

# Live NeuronCore utilization monitor (equivalent to nvidia-smi)
neuron-top
```

Expected `neuron-ls` output:

```
+--------+------------+---------+--------+
| Name   | NeuronCore | Memory  | Status |
+--------+------------+---------+--------+
| NC0    | 0,1,2,3    | 32 GB   | OK     |
+--------+------------+---------+--------+
```

### 4.4 Verify All Required Packages

Run the following one-shot check:

```bash
python3 - <<'EOF'
packages = [
    "torch", "torch_neuronx", "torch_xla",
    "neuronx_distributed", "transformers", "accelerate", "vllm"
]
for pkg in packages:
    try:
        m = __import__(pkg)
        ver = getattr(m, "__version__", "installed, no version attr")
        print(f"  [OK]      {pkg}: {ver}")
    except ImportError as e:
        print(f"  [MISSING] {pkg}: {e}")
EOF
```

**Expected output** (DeprecationWarnings on `neuronx_distributed` are normal and can be ignored):

```
  [OK]      torch: 2.9.0+cu128
  [OK]      torch_neuronx: 2.9.0.2.11.19912+e48cd891
  [OK]      torch_xla: 2.9.0
  [OK]      neuronx_distributed: installed, no version attr
  [OK]      transformers: 4.56.2
  [OK]      accelerate: 1.13.0
  [OK]      vllm: 0.13.0
```

{% hint style="warning" %}
**Do not** `pip install vllm` or `git clone vllm-neuron`. The correct version is already pre-installed in the image. Reinstalling may break the Neuron plugin.
{% endhint %}

## 5. Using the Prestarted vLLM Service

> Reference: [Quickstart: Serve models online with vLLM on Neuron](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/libraries/nxd-inference/vllm/quickstart-vllm-online-serving.html)

{% hint style="warning" %}
On Yotta Trainium pods, a vLLM server is already started automatically.\
Do NOT run `vllm serve` again unless you are sure no service is running.
{% endhint %}

### 5.1 Check if vLLM is already running

Run:

```bash
curl http://127.0.0.1:8080/v1/models
```

If you see a response like:

```bash
{
  "object": "list",
  "data": [
    {
      "id": "TinyLlama/TinyLlama-1.1B-Chat-v1.0"
    }
  ]
}
```

Then the service is already running.

### 5.2 Confirm the active model

The active model is:

```
TinyLlama/TinyLlama-1.1B-Chat-v1.0
```

### 5.3 Run inference (curl)

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "TinyLlama/TinyLlama-1.1B-Chat-v1.0",
    "messages": [
      {"role": "user", "content": "Hello, who are you?"}
    ],
    "max_tokens": 64
  }'
```

## Quick Reference

```bash
# Activate conda (required after every new SSH session)
source /opt/conda/bin/activate

# Check Neuron devices
neuron-ls

# Live NeuronCore monitor
neuron-top

# Set required env vars
export VLLM_NEURON_FRAMEWORK="neuronx-distributed-inference"
export NEURON_COMPILED_ARTIFACTS=~/neuron_compiled_models/tinyllama   # use ~ not /workspace
export HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxx   # only needed for gated models (e.g. Llama-3.1)

# Health check
curl http://localhost:8000/health

# View server logs
tail -f /workspace/vllm.log

# List loaded models
curl http://localhost:8000/v1/models | python3 -m json.tool

# Clear compiled artifacts (free disk space)
rm -rf ~/neuron_compiled_models

# View server logs (background mode)
tail -f ~/vllm.log
```

## Verified Package Versions

| Package               | Version                            |
| --------------------- | ---------------------------------- |
| `torch`               | 2.9.0+cu128                        |
| `torch_neuronx`       | 2.9.0.2.11.19912                   |
| `torch_xla`           | 2.9.0                              |
| `neuronx_distributed` | (installed, no `__version__`)      |
| `transformers`        | 4.56.2                             |
| `accelerate`          | 1.13.0                             |
| `vllm`                | 0.13.0 (with `vllm-neuron` plugin) |

> Reference: [AWS Neuron Docs — vLLM on Neuron](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/libraries/nxd-inference/vllm/index.html)
