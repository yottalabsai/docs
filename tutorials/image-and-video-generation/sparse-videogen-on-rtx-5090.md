# Sparse-VideoGen on RTX 5090

Based on [Sparse VideoGen2 (Xi et al., NeurIPS 2025 Spotlight)](https://arxiv.org/abs/2505.18875).

Sparse VideoGen 2 (SVG2) is a training-free inference acceleration framework for video diffusion transformers. Rather than changing model weights, it exploits the inherent sparsity in 3D full attention — identifying which tokens actually matter via semantic-aware sparse attention and flash k-means clustering — to deliver roughly **2× end-to-end speedup** with minimal visual quality loss. It supports HunyuanVideo and Wan 2.1 (T2V and I2V), all of which fit on a single RTX 5090.

***

### Deploy a Pod

Log in to the [Yotta Labs Console](https://console.yottalabs.ai/), go to **Compute → Pods**, and deploy a new Pod with the following settings:

| Setting       | Value                 |
| ------------- | --------------------- |
| GPU           | RTX 5090              |
| Template      | `pytorch` (CUDA 12.8) |
| System Volume | 150 GB minimum        |

Once the Pod is `Running`, click **Connect** and open a terminal via **File → New → Terminal** in JupyterLab.

***

### Environment Setup

Clone the repo with `GIT_LFS_SKIP_SMUDGE=1` to skip the large demo assets:

```bash
cd ~
GIT_LFS_SKIP_SMUDGE=1 git clone https://github.com/svg-project/Sparse-VideoGen.git
cd ~/Sparse-VideoGen
```

Create the conda environment and install the base package. SVG2's `pyproject.toml` uses hatchling with editable installs, so `editables` needs to be present before anything else:

```bash
conda create -n SVG python=3.12.9 -y
conda activate SVG

python -m ensurepip --upgrade
python -m pip install --upgrade pip

pip install editables hatchling
pip install -e .
pip install flash-attn --no-build-isolation
```

Install the pinned versions of diffusers and transformers. SVG2 requires `diffusers==0.34.0`, and that version is only compatible with `transformers<5.0` — specifically `transformers==4.49.0`. Installing either package at the wrong version will cause import errors at runtime:

```bash
pip install "diffusers==0.34.0" "transformers==4.49.0"
```

Install the remaining runtime dependencies:

```bash
pip install termcolor imageio imageio-ffmpeg opencv-python einops \
  sentencepiece protobuf accelerate
```

Pull down the git submodules. Cutlass is large so this may take a few minutes:

```bash
cd ~/Sparse-VideoGen
git submodule update --init --recursive

# Verify — all three should be populated
ls svg/kernels/3rdparty/
# Expected: cutlass  flashinfer  pybind
```

Build the customized attention kernels. The conda environment injects its own C++ compiler via `NVCC_PREPEND_FLAGS`, which breaks CUDA header detection — unset it and point cmake explicitly at the system CUDA installation before building:

```bash
export CUDA_HOME=/usr/local/cuda-12.8
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
export CUDAHOSTCXX=/usr/bin/g++
unset NVCC_PREPEND_FLAGS

pip install -U setuptools cmake

cd ~/Sparse-VideoGen/svg/kernels
rm -rf build && mkdir -p build && cd build

cmake \
  -DCMAKE_PREFIX_PATH="$(python -c 'import torch; print(torch.utils.cmake_prefix_path)')" \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.8/bin/nvcc \
  -DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda-12.8 \
  -DCUDA_INCLUDE_DIRS=/usr/local/cuda-12.8/include \
  -DUSE_SYSTEM_NVTX:BOOL=ON \
  ..

make -j$(nproc)
cd ~/Sparse-VideoGen
```

Install FlashInfer from the submodule. The correct path is `svg/kernels/3rdparty/flashinfer` — not `3rdparty/flashinfer` as listed in the upstream README:

```bash
cd ~/Sparse-VideoGen/svg/kernels/3rdparty/flashinfer
pip install --no-build-isolation --verbose --editable .
cd ~/Sparse-VideoGen
```

Finally, install cuVS:

```bash
pip install cuvs-cu12 --extra-index-url=https://pypi.nvidia.com
```

> You may see dependency conflict warnings about `cuda-toolkit` and `nvidia-nvjitlink-cu12` from `libcuvs`. These are warnings, not errors, and do not affect inference.

***

### Download Model Checkpoints

Set your Hugging Face token:

```bash
export HF_TOKEN="your_token_here"
```

**Wan 2.1 T2V** (\~54 GB). Use the official `Wan-AI` repo — other forks may have incompatible checkpoint formats:

```bash
hf download Wan-AI/Wan2.1-T2V-14B-Diffusers \
  --local-dir ~/Sparse-VideoGen/ckpts/Wan2.1-T2V-720P \
  --token $HF_TOKEN
```

Verify the transformer directory contains a shard index file:

```bash
ls ~/Sparse-VideoGen/ckpts/Wan2.1-T2V-720P/transformer/
# Should include: diffusion_pytorch_model.safetensors.index.json
```

**Wan 2.1 I2V**:

```bash
hf download Wan-AI/Wan2.1-I2V-14B-720P-Diffusers \
  --local-dir ~/Sparse-VideoGen/ckpts/Wan2.1-I2V-720P \
  --token $HF_TOKEN
```

**HunyuanVideo**:

```bash
hf download tencent/HunyuanVideo \
  --local-dir ~/Sparse-VideoGen/ckpts/HunyuanVideo \
  --token $HF_TOKEN
```

The inference scripts default to loading models directly from HuggingFace by repo ID. Update them to use your local paths instead:

```bash
sed -i 's|Wan-AI/Wan2.1-T2V-14B-Diffusers|/home/user/Sparse-VideoGen/ckpts/Wan2.1-T2V-720P|' \
  ~/Sparse-VideoGen/scripts/wan/wan_t2v_720p_sap.sh

sed -i 's|Wan-AI/Wan2.1-I2V-14B-720P-Diffusers|/home/user/Sparse-VideoGen/ckpts/Wan2.1-I2V-720P|' \
  ~/Sparse-VideoGen/scripts/wan/wan_i2v_720p_sap.sh

sed -i 's|tencent/HunyuanVideo|/home/user/Sparse-VideoGen/ckpts/HunyuanVideo|' \
  ~/Sparse-VideoGen/scripts/hyvideo/hyvideo_t2v_720p_sap.sh
```

Verify:

```bash
grep model_id ~/Sparse-VideoGen/scripts/wan/wan_t2v_720p_sap.sh
grep model_id ~/Sparse-VideoGen/scripts/wan/wan_i2v_720p_sap.sh
grep model_id ~/Sparse-VideoGen/scripts/hyvideo/hyvideo_t2v_720p_sap.sh
```

***

### Run Video Generation

The default prompt is loaded from `examples/1/prompt.txt`. Edit it to change the prompt, or change `prompt_id` in the script to use a different example. Always run from the project root.

**Wan 2.1 Text-to-Video:**

Default prompts:arrow\_down\_small:

```
==================== Prompts ====================
Prompt: warm colors dominate the room, with a focus on the tabby cat sitting contently in the center. the scene captures the fluffy orange tabby cat wearing a tiny virtual reality headset. the setting is a cozy living room, adorned with soft, warm lighting and a modern aesthetic. a plush sofa is visible in the background, along with a few lush potted plants, adding a touch of greenery. the cat's tail flicks curiously, as if engaging with an unseen virtual environment. its paws swipe at the air, indicating a playful and inquisitive nature, as it delves into the digital realm. the atmosphere is both whimsical and futuristic, highlighting the blend of analog and digital experiences.
Negative Prompt: Bright tones, overexposed, static, blurred details, subtitles, style, works, paintings, images, static, overall gray, worst quality, low quality, JPEG compression residue, ugly, incomplete, extra fingers, poorly drawn hands, poorly drawn faces, deformed, disfigured, misshapen limbs, fused fingers, still picture, messy background, three legs, many people in the background, walking backwards
```

```bash
cd ~/Sparse-VideoGen
bash scripts/wan/wan_t2v_720p_sap.sh
```

**Wan 2.1 Image-to-Video:**

Default prompts:arrow\_down\_small:

```
==================== Prompts ====================
Prompt: warm colors dominate the room, with a focus on the tabby cat sitting contently in the center. the scene captures the fluffy orange tabby cat wearing a tiny virtual reality headset. the setting is a cozy living room, adorned with soft, warm lighting and a modern aesthetic. a plush sofa is visible in the background, along with a few lush potted plants, adding a touch of greenery. the cat's tail flicks curiously, as if engaging with an unseen virtual environment. its paws swipe at the air, indicating a playful and inquisitive nature, as it delves into the digital realm. the atmosphere is both whimsical and futuristic, highlighting the blend of analog and digital experiences.
Negative Prompt: Bright tones, overexposed, static, blurred details, subtitles, style, works, paintings, imag
```

```bash
cd ~/Sparse-VideoGen
bash scripts/wan/wan_i2v_720p_sap.sh
```

**HunyuanVideo Text-to-Video:**

Default prompts:arrow\_down\_small:

```
==================== Prompts ====================
A plush teddy bear, with soft brown fur and a red bow tie, sits on a lush green lawn under a bright, sunny sky. Nearby, a vibrant blue frisbee lies on the grass, hinting at playful moments. The scene transitions to the teddy bear being gently tossed into the air, its limbs flailing joyfully, as the frisbee soars in the background. The bear lands softly, surrounded by daisies, while the frisbee spins to a stop beside it. Finally, the teddy bear is propped up against a tree trunk, holding the frisbee in its lap, creating a heartwarming image of companionship and play.
```

```bash
cd ~/Sparse-VideoGen
bash scripts/hyvideo/hyvideo_t2v_720p_sap.sh
```

When running correctly, you should see `Attention processors replaced with SAP pattern.` followed by `Centroids initialized at layer N` as the k-means attention warms up. Output videos are saved under `result/` with a nested directory structure that encodes the generation config.

***

### View Output Videos

```bash
find ~/Sparse-VideoGen/result -name "*.mp4" | sort
```

To play them inline in a JupyterLab notebook cell:

```python
import glob, os
from IPython.display import Video, display

results_dir = os.path.expanduser("~/Sparse-VideoGen/result")
videos = sorted(glob.glob(f"{results_dir}/**/*.mp4", recursive=True))

if not videos:
    print("No output videos found.")
else:
    for v in videos:
        print(f"📹 {v}")
        display(Video(v, embed=True, width=640))
```

> If the cell hangs on large files, interrupt the kernel and open the video directly from the JupyterLab file browser instead.

<figure><img src="../../.gitbook/assets/image (64).png" alt=""><figcaption></figcaption></figure>

{% file src="../../.gitbook/assets/1-0.mp4" %}

***

### Want to Use Your Own Prompt/Image?

Each inference script loads its prompt from a text file under `examples/`. The file used is determined by the `prompt_id` variable at the top of the script:

| Script                    | Default `prompt_id` | Prompt file             | Image file             |
| ------------------------- | ------------------- | ----------------------- | ---------------------- |
| `wan_t2v_720p_sap.sh`     | 1                   | `examples/1/prompt.txt` | —                      |
| `wan_i2v_720p_sap.sh`     | 1                   | `examples/1/prompt.txt` | `examples/1/image.jpg` |
| `hyvideo_t2v_720p_sap.sh` | 7                   | `examples/7/prompt.txt` | —                      |

You can also confirm which prompt is being used at runtime — it is printed to the terminal at the start of each run.

To use your own prompt, create a new example directory and write your prompt into it:

```bash
mkdir -p ~/Sparse-VideoGen/examples/8
echo "your prompt here" > ~/Sparse-VideoGen/examples/8/prompt.txt
```

For I2V, also provide an input image:

```bash
cp /path/to/your/image.jpg ~/Sparse-VideoGen/examples/8/image.jpg
```

Then open the script and change `prompt_id` to match your new directory:

```bash
nano ~/Sparse-VideoGen/scripts/wan/wan_t2v_720p_sap.sh
# Change: prompt_id=1
# To:     prompt_id=8
```

Run as usual and the output filename will reflect your `prompt_id`, e.g. `8-0.mp4`.

***

### Additional Resources

* [Sparse-VideoGen GitHub](https://github.com/svg-project/Sparse-VideoGen)
* [SVG2 Paper (arXiv)](https://arxiv.org/abs/2505.18875)
* [SVG1 Paper (arXiv)](https://arxiv.org/abs/2502.01776)
* [Project Page](https://svg-project.github.io/)
* [Yotta Labs Console](https://console.yottalabs.ai/)
