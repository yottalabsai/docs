# Quant-VideoGen on RTX 5090

**Based on** &#x20;

```latex
@article{xi2026quant,
  title={Quant VideoGen: Auto-Regressive Long Video Generation via 2-Bit KV-Cache Quantization},
  author={Xi, Haocheng and Yang, Shuo and Zhao, Yilong and Li, Muyang and Cai, Han and Li, Xingyang and Lin, Yujun and Zhang, Zhuoyang and Zhang, Jintao and Li, Xiuyu and others},
  journal={arXiv preprint arXiv:2602.02958},
  year={2026}
}
```

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

<figure><img src="../../.gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>

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
