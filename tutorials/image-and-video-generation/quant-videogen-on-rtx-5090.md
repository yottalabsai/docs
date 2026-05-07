# Quant-VideoGen on RTX 5090

**Based on** _Quant-VideoGen:Auto-Regressive Long Video Generation via 2-Bit KV-Cache Quantization_ by Xi et al. (2026).

**ArXiv Link:** [https://arxiv.org/abs/2602.02958v1](https://arxiv.org/abs/2602.02958v1)

**GitHub:** [svg-project/Quant-VideoGen](https://github.com/svg-project/Quant-VideoGen)

### Why Quant VideoGen?&#x20;

Autoregressive video generation models face a critical memory bottleneck that severely limits long video synthesis.&#x20;

Picture this: generating just 5 seconds of 480p video demands approximately 34GB of memory for KV cache storage alone—already exceeding what a single RTX 5090 can handle.&#x20;

Why is this such a problem? These models generate frames sequentially, and here's the catch: _each new frame must maintain complete references to all previously generated frames_. This means memory consumption grows linearly with every additional frame added, quickly overwhelming even the most powerful consumer hardware.

However,there is an improvement we can apply--quantization. While quantization for videos are very different from textual ones.

**Quantization in Text Models (Why It Works):**

Traditional quantization works straightforwardly. In text models like BERT, KV cache values are distributed **uniformly**. For example:

```
Sentence: "The cat is on the mat"

Token 1 (The):  [-0.5, 0.3, 1.2, -0.8, 0.6, ...]
Token 2 (cat):  [-0.4, 0.2, 1.1, -0.7, 0.5, ...]
Token 3 (is):   [-0.6, 0.4, 1.3, -0.9, 0.7, ...]
...

Observation: All values stay within [-1, 1.5]—nicely uniform!
```

**Why does it work for text?**&#x20;

Because the value range is fixed and uniform across all tokens—no extreme outliers, all tokens treated equally.

***

**Quantization in Video Models (Why It Fails):**

Video KV cache exhibits **wildly non-uniform** value distributions. Within a single frame:

```
Static background (sky):        [0.1, 0.05, -0.02, 0.08, ...]
High-contrast edges (objects):  [50.2, 48.9, 52.1, 49.5, ...]
Fast-moving regions:            [1200.5, 1150.2, 1300.1, ...]

Result: Value range spans from 0.05 to 1300—a 26,000× difference!
```

Applying traditional text quantization causes **catastrophic failure**:

```
1. min = 0.05, max = 1300
2. Scale factor = (1300 - 0.05) / 255 ≈ 5.09

Quantizing background value 0.1:
  (0.1 - 0.05) / 5.09 ≈ 0.01 → rounds to 0 (8-bit integer)
  Dequantization: 0 * 5.09 + 0.05 = 0.05
  Original was 0.1, now we get 0.05 → 50% precision loss!

Quantizing motion value 1200:
  (1200 - 0.05) / 5.09 ≈ 236 (acceptable)
  Dequantization: 236 * 5.09 + 0.05 ≈ 1201 (reasonable)
```

**The root cause:** Because of the extreme outlier (1300), the entire quantization range stretches to accommodate it. This forces most normal values (0.1-50 range) into coarse bins, destroying precision.

```
Text Model KV Cache Distribution (Uniform):
 ━━━━━━━━━━━━━━━━━━
  -1      0     1.5
   ▁▂▃▄▅▆▇█▇▆▅▄▃▂▁
   → 8-bit quantization allocates resolution fairly to all values

Video Model KV Cache Distribution (Extremely Non-Uniform):
━┃━━━━━━━━━━━━━━━━━┃━━━━━━━━━━━━━━━
0.05              50              1300
▂▁▁▁▁░░░░░░░░░░░░░░░░░░░░░░░░░▂▃▄▅▆▇█
↑ Massive concentration here    ↑ Single outlier
→ 8-bit quantization wastes resolution on sparse outliers,
  leaving dense regions with almost no precision!
```

To address this challenge, researchers from UC Berkeley, MIT, NVIDIA, Amazon, and University of Texas at Austin introduce Quant VideoGen, an innovative framework that exploits the inherent spatial-temporal redundancy in video content. The key insight is that adjacent video frames and spatially nearby regions exhibit high similarity, enabling aggressive compression.

<figure><img src="../../.gitbook/assets/image (35).png" alt=""><figcaption><p>Figure from <a href="https://arxiv.org/abs/2602.02958">Xi et al., 2026</a></p></figcaption></figure>

The solution comprises two main components:&#x20;

(1) **Semantic-Aware Smoothing** groups similar tokens using k-means clustering, quantizing residuals relative to cluster centroids rather than raw values. This reduces key cache quantization error by 6.9× and value cache error by 2.6×.&#x20;

(2) **Progressive Residual Quantization** applies the smoothing process iteratively across multiple stages, capturing semantic information at different granularities while maintaining uniform value distributions suitable for low-precision quantization.

Experimental results demonstrate exceptional performance across multiple state-of-the-art models (LongCat-Video, HY-WorldPlay, Self-Forcing). QVG achieves 6.94-7.05× compression ratios while maintaining near-lossless visual quality (PSNR >28.7). Crucially, end-to-end latency overhead remains below 4%, making the approach practical for real applications.

This breakthrough dramatically lowers hardware barriers—enabling long video generation on consumer-grade GPUs that previously required enterprise-level hardware, while maintaining sub-4% performance overhead.



### **How to run Quant-VideoGen on a single RTX 5090**

### Prerequisites

* A [Yotta Labs](https://www.yottalabs.ai/) account with sufficient credits
* A [Hugging Face](https://huggingface.co/) account and access token (`HF_TOKEN`)
* Familiarity with basic Python and terminal commands
* 🐱 **LongCat-Video** — best starting point, smallest VRAM footprint
* ⚡ **Self-Forcing** — streaming video generation
* 🌍 **HY-WorldPlay** — world model video generation

| Model         | BF16 Total KV | INT2 Total KV | Compression |
| ------------- | ------------- | ------------- | ----------- |
| LongCat-Video | 22,272 MB     | 3,231 MB      | **6.9×**    |
| Self-Forcing  | 46,073 MB     | 6,614 MB      | **7.0×**    |
| HY-WorldPlay  | 29,700 MB     | 4,235 MB      | **7.0×**    |

> **Why RTX 5090?** Self-Forcing's BF16 KV cache alone requires \~46 GB — impossible on a single 32 GB GPU. QVG brings it down to \~6.6 GB, making long video generation fully feasible on RTX 5090.

**Resources:** [GitHub](https://github.com/svg-project/Quant-VideoGen) · [Project Page](https://svg-project.github.io/qvg/) · [arXiv](https://arxiv.org/abs/2602.02958)

***

### Prerequisites

* A [Yotta Labs](https://www.yottalabs.ai/) account with sufficient credits
* A [Hugging Face](https://huggingface.co/) account and access token with read permissions
* Basic familiarity with Python and the terminal

#### GPU Requirements

| GPU          | VRAM      | Notes                             |
| ------------ | --------- | --------------------------------- |
| **RTX 5090** | **32 GB** | ✅ Recommended on Yotta Labs       |
| RTX 4090     | 24 GB     | Feasible for LongCat-Video only   |
| H100 / A100  | 80 GB     | Can run BF16 baseline without QVG |

***

### Step 1 — Deploy a Pod

1. Log in to the [Yotta Labs Console](https://console.yottalabs.ai/).
2. Navigate to **Compute → Pods** and click **Deploy**.
3. Select **RTX 5090** as the GPU.
4. Under **Pod Template**, choose `pytorch`.
5. Set **System Volume** to at least **150 GB** (checkpoints vary: LongCat \~15 GB, Self-Forcing \~30 GB, HY-WorldPlay \~25 GB).
6. Click **Deploy** and wait for the Pod to reach `Running` state.

***

### Step 2 — Open JupyterLab

1. Click **Connect** on the Pod card.
2. Click the link for **port 8888** to open JupyterLab.
3. Create a new notebook: **File → New → Notebook**, select the default kernel.
4. For steps marked **\[Terminal]**, open a terminal via **File → New → Terminal**.

> Steps below are clearly marked as either **\[Terminal]** or **\[Notebook Cell]**.

***

### Step 3 — Set Up the Environment

#### 3.1 — Clone the Repository

**\[Terminal]**

```bash
cd ~
git clone https://github.com/svg-project/Quant-VideoGen.git
```

#### 3.2 — Create a Conda Environment

```bash
#!/bin/bash
set -e  

echo "📦 Initializing conda..."
conda init bash
source ~/.bashrc
conda create -n qvg python=3.12.9 -y
conda activate qvg
```

#### 3.3 — Install QVG and Dependencies

**\[Terminal]** — Run inside the `qvg` environment.

```bash
cd ~/Quant-VideoGen

pip install uv
pip install huggingface_hub        # provides the huggingface-cli command

# Install QVG with all three backend extras
uv pip install -e ".[all]"
```

> If you only need one backend, install just that extra — e.g. `uv pip install -e ".[longcat]"`.

#### 3.4 — Install Flash Attention

```bash
uv pip install \
  https://github.com/Dao-AILab/flash-attention/releases/download/v2.8.3/flash_attn-2.8.3+cu12torch2.8cxx11abiFALSE-cp312-cp312-linux_x86_64.whl
```

> If this wheel doesn't match your environment, check your versions first:
>
> ```bash
> python -c "import torch; print(torch.__version__, torch.version.cuda)"
> ```
>
> Then find the matching wheel at the [flash-attention releases page](https://github.com/Dao-AILab/flash-attention/releases).

***

### Step 4 — Download Model Checkpoints

When downloading check points, the official repository provides three choices: LongCat-video, Self- forcing and HY-WorldPlay.&#x20;

Quant-VideoGen supports three backends with very different hardware requirements.&#x20;

1. **Self-Forcing** is the most lightweight option, built on Wan2.1-T2V-1.3B (\~5 GB of weights), and runs comfortably on a single RTX 5090 (32 GB VRAM).&#x20;
2. **HY-WorldPlay** uses a mid-sized model (\~25 GB of weights) and fits within a single RTX 5090 with QVG INT2 quantization applied to the KV cache.
3. &#x20;**LongCat-Video** is the most memory-intensive backend — it uses Mistral-Small-24B as its text encoder (22 GB) paired with a large DiT (51 GB), totaling \~83 GB of model weights. Since QVG only quantizes the inference-time KV cache and not the model weights themselves, LongCat-Video cannot be loaded onto a single 32 GB GPU regardless of quantization settings. It requires at least **2×H200 (160 GB total VRAM)** to run.

**We start from self-forcing video generation with Wan2.1-T2V-1.3B on a single RTX5090 GPU.**

| Backend       | Model                   | Weights | Min GPU     |
| ------------- | ----------------------- | ------- | ----------- |
| Self-Forcing  | Wan2.1-T2V-1.3B + DMD   | \~5 GB  | 1× RTX 5090 |
| HY-WorldPlay  | —                       | \~25 GB | 1× RTX 5090 |
| LongCat-Video | Mistral-Small-24B + DiT | \~83 GB | 2× H200     |

```bash
cd ~/Quant-VideoGen
export HF_TOKEN="your_token_here"   # <-- replace with your HF token

bash scripts/Self-Forcing/download_models.sh
```

Verify the downloaded files:

```bash
du -sh ~/Quant-VideoGen/ckpts/Self-Forcing/*
```

Expected output :

```
17G     /home/user/Quant-VideoGen/ckpts/Self-Forcing/Wan2.1-T2V-1.3B
5.3G    /home/user/Quant-VideoGen/ckpts/Self-Forcing/self_forcing_dmd.pt
```

***

### Step 5 — Run Video Generation

```bash
cd ~/Quant-VideoGen
bash scripts/Self-Forcing/run_qvg.sh
```

Output videos are saved to `~/Quant-VideoGen/results/`.

<figure><img src="../../.gitbook/assets/image (47).png" alt="" width="508"><figcaption></figcaption></figure>

***

### Step 6 — View Output Videos

**\[ In Notebook cell ]**

```python
import glob, os
from IPython.display import Video, display

results_dir = os.path.expanduser("~/Quant-VideoGen/results")
videos = sorted(glob.glob(f"{results_dir}/**/*.mp4", recursive=True))

if not videos:
    print("No output videos found yet. Run a generation cell above first.")
else:
    for v in videos:
        print(f"📹 {v}")
        display(Video(v, embed=True, width=640))
```

***

### Step 7 — Want to Customize Quantization Settings?

Quantization options are defined inside each `run_qvg.sh` script. Key parameters:

| Parameter       | Default                      | Description                                                     |
| --------------- | ---------------------------- | --------------------------------------------------------------- |
| `quant_method`  | `triton-nstages-kmeans-int2` | Quantization algorithm. INT2 gives \~7× compression.            |
| `block_size`    | `64`                         | Token block size. Smaller = finer-grained but slower.           |
| `num_centroids` | `256`                        | K-means centroids. More = better quality, slightly more memory. |

***

## Larger model & Further explorations

> **GPU Requirements**
>
> * **LongCat-Video** — requires **2× H200** (model weights total \~83 GB: Mistral-Small-24B text encoder + large DiT)
> * **HY-WorldPlay** — runs on a **single RTX 5090** (32 GB); uses `--offload_text_encoder` to stay within VRAM budget

***

### &#x20;LongCat-Video (2× H200)

LongCat-Video generates long videos auto-regressively by extending from an initial base clip. The workflow has three sequential steps: generate a base clip → extend with BF16 → extend with QVG INT2 quantization.

#### Step 1 — Download Checkpoints

```bash
cd ~/Quant-VideoGen
huggingface-cli download meituan-longcat/LongCat-Video \
  --local-dir ckpts/LongCat-Video \
  --token $HF_TOKEN
```

Verify:

```bash
du -sh ~/Quant-VideoGen/ckpts/LongCat-Video/*
```

#### Step 2 — Generate Base Clip

The base clip (`results/longcat/base/1-0.mp4`) is required by both `run_bf16.sh` and `run_qvg.sh` as the starting frame. Always run this first.

```bash
cd ~/Quant-VideoGen
bash scripts/LongCat/base.sh
```

Confirm the output exists before proceeding:

```bash
ls results/longcat/base/
```

#### Step 3 — Extend with BF16 (quality reference)

```bash
bash scripts/LongCat/run_bf16.sh
```

Output: `results/longcat/bf16/`

#### Step 4 — Extend with QVG INT2 (recommended)

QVG reduces the KV cache by \~7× via 2-bit quantization, significantly cutting memory during the long auto-regressive extension.

```bash
bash scripts/LongCat/run_qvg.sh
```

Output: `results/longcat/triton-nstages-kmeans-int2_64/.../`

#### View Output in JupyterLab Notebook

Open a new notebook cell and run:

```python
import glob, os
from IPython.display import Video, display

#switch to the right directoty
import os
os.chdir('/home/user')
print(os.getcwd())
results_dir = os.path.expanduser("Quant-VideoGen/results/longcat")
videos = sorted(glob.glob(f"{results_dir}/**/*.mp4", recursive=True))

if not videos:
    print("No output videos found.")
else:
    for v in videos:
        print(f"📹 {v}")
        display(Video(v, embed=True, width=640))
```

***

### &#x20;HY-WorldPlay (1× RTX 5090)

HY-WorldPlay is an image-to-video world model — it takes a single input image and a camera movement sequence (`--pose`) to generate a navigable video. It uses `--offload_text_encoder` to keep the text encoder on CPU and fit within 32 GB VRAM.

#### Step 1 — Download Checkpoints

```bash
cd ~/Quant-VideoGen
bash scripts/HY-WorldPlay/download_models.sh
```

Verify:

```bash
du -sh ~/Quant-VideoGen/ckpts/HY-WorldPlay/*
```

#### Step 2 — Run with QVG INT2 (recommended)

```bash
cd ~/Quant-VideoGen
bash scripts/HY-WorldPlay/run_qvg.sh
```

Output: `results/hyworldplay/triton-nstages-kmeans-int2_64/.../`

The default input image is `assets/hyworld.png` and the default prompt describes a stone arch bridge scene. To use your own image and prompt, edit the script directly:

```bash
nano scripts/HY-WorldPlay/run_qvg.sh
```

Change these two lines:

```bash
PROMPT='your prompt here'
IMAGE_PATH=path/to/your/image.png
```

You can also customize the camera trajectory via `--pose`. The default is:

```
w-8,s-8,a-8,d-8,up-8,down-8
```

Each entry is a direction followed by number of frames: `w` = forward, `s` = backward, `a` = left, `d` = right, `up` = tilt up, `down` = tilt down.

#### Step 3 — Run BF16 baseline (optional, quality comparison)

bash

```bash
bash scripts/HY-WorldPlay/run_bf16.sh
```

> Unlike LongCat, HY-WorldPlay does **not** require a base clip first — `run_bf16.sh` and `run_qvg.sh` are both standalone.

#### View Output in JupyterLab Notebook

python

```python
import glob, os
from IPython.display import Video, display

#switch to the right directoty
import os
os.chdir('/home/user')
print(os.getcwd())
results_dir = os.path.expanduser("Quant-VideoGen/results/hyworldplay")
videos = sorted(glob.glob(f"{results_dir}/**/*.mp4", recursive=True))

if not videos:
    print("No output videos found.")
else:
    for v in videos:
        print(f"📹 {v}")
        display(Video(v, embed=True, width=640))
```

***

### Troubleshooting

* **LongCat: `IndexError: list index out of range`** The base clip is missing. Run `bash scripts/LongCat/base.sh` first before running `run_bf16.sh` or `run_qvg.sh`.
* **LongCat: `CUDA out of memory` on H200** Make sure you have 2× H200. The model weights alone total \~83 GB and cannot be split across GPUs with the current single-process setup.
* **HY-WorldPlay: `CUDA out of memory` on RTX 5090** Confirm `--offload_text_encoder` is present in the script. Check with:

```bash
grep offload scripts/HY-WorldPlay/run_qvg.sh
```
