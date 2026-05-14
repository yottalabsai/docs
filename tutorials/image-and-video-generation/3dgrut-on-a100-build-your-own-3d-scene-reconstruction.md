# 3DGRUT on A100: Build your own 3D Scene Reconstruction

> Created based on [https://github.com/nv-tlabs/3dgrut](https://github.com/nv-tlabs/3dgrut)\
> &#xNAN;_&#x4E;VIDIA 3DGRUT · SIGGRAPH Asia 2024 / CVPR 2025_

***

### 1. What is 3DGRUT?

#### Big Picture :arrow\_down:

**3DGRUT** is NVIDIA's official open-source codebase implementing two groundbreaking 3D scene reconstruction and rendering methods:

* **3DGRT** — _3D Gaussian Ray Tracing_ (SIGGRAPH Asia 2024, Journal Track)
* **3DGUT** — _3D Gaussian Unscented Transform_ (CVPR 2025, **Oral**)

Both methods build on top of the classic **3D Gaussian Splatting (3DGS)** paradigm — representing a scene as millions of tiny 3D Gaussian "blobs" — but they push the technology far beyond what rasterization-based 3DGS can do.

#### 3DGRT vs. Classic 3DGS

| Feature               | Classic 3DGS                    | 3DGRT (3DGRUT)                  |
| --------------------- | ------------------------------- | ------------------------------- |
| Rendering method      | Rasterization (tile-based sort) | Ray tracing via GPU RT cores    |
| Shadows & reflections | ❌ No                            | ✅ Yes (secondary rays)          |
| Distorted cameras     | ❌ Limited                       | ✅ Full support                  |
| Rolling shutter       | ❌ No                            | ✅ Yes                           |
| Speed vs. 3DGS        | Faster                          | Slightly slower, but far richer |
| Hardware requirement  | Any CUDA GPU                    | NVIDIA RT-core GPU (Turing+)    |

#### How 3DGRT Works Under the Hood

Unlike classic 3DGS which rasterizes Gaussians by projecting them onto screen-space tiles and sorting them front-to-back, **3DGRT performs ray tracing** — it:

1. Wraps each Gaussian particle in a **bounding mesh primitive**
2. Inserts all bounding meshes into an **OptiX BVH** (Bounding Volume Hierarchy)
3. For each pixel, casts a ray and traverses the BVH in O(log n)
4. Shades **batches of intersected Gaussians** in depth order
5. Optionally fires **secondary rays** from surface hits for reflections, shadows, and refractions

```
Input images → COLMAP SfM → Sparse 3D points
     ↓
Initialize 3D Gaussians (position, covariance, opacity, SH color)
     ↓
Build OptiX BVH over Gaussian bounding meshes
     ↓
Ray-trace per pixel → accumulate Gaussian contributions
     ↓
Compare render vs. ground truth → backprop → update Gaussians
     ↓
Densification (clone + split) every 100 iterations up to ~15k steps
     ↓
Final 3D scene: 1–5 million Gaussians, real-time ray-traced rendering
```

#### What is 3DGUT?

3DGUT (CVPR 2025 Oral) solves a different problem: making Gaussian Splatting work with **highly distorted cameras** — fish-eye lenses, rolling-shutter sensors, and time-dependent camera models common in robotics and autonomous driving. It uses the **Unscented Transform** to propagate Gaussian distributions through non-linear camera projections, enabling a hybrid rasterizer that's both fast and accurate for distorted optics.

> **Quick rule of thumb:**
>
> * Standard perspective cameras + want reflections/shadows → use **3DGRT**
> * Distorted cameras (fish-eye, rolling shutter) or need max speed → use **3DGUT**

***

### 2. Spin up a Yotta Labs Pod

#### Recommended GPU

For 3DGRUT you **must** have an NVIDIA GPU with RT cores (Turing architecture or newer). My recommendation:

| GPU                | VRAM  | Best for                                    |
| ------------------ | ----- | ------------------------------------------- |
| **H100 SXM5 80GB** | 80 GB | Best all-around: training + 3DGRT rendering |
| A100 80GB          | 80 GB | Great for training, RT cores present        |
| RTX 5090           | 32 GB | Smaller scenes, fastest RT throughput       |

> ⚠️ **Important:** 3DGRT _requires_ RT cores for fast ray traversal. Without them it falls back to software ray tracing, which is \~10× slower. Always pick an Turing+ (RTX/A-series/H-series) GPU.

1. Log in to the [Yotta Labs Console](https://console.yottalabs.ai/).
2. Navigate to **Compute → Pods** and click **Deploy**.
3. Select **RTX 5090** as the GPU.
4. Under **Pod Template**, choose `pytorch`.
5. Set **System Volume** to at least **150 GB** (checkpoints vary: LongCat \~15 GB, Self-Forcing \~30 GB, HY-WorldPlay \~25 GB).
6. Click **Deploy** and wait for the Pod to reach `Running` state.

***

### 3. Install & Build 3DGRUT

{% stepper %}
{% step %}
### Install Miniconda

Since most VMs don't come with `conda` pre-installed, install Miniconda to your home directory — no root access needed:

```bash
cd ~
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -b -p ~/miniconda3
~/miniconda3/bin/conda init bash
source ~/.bashrc
```

Verify the installation:

```bash
conda --version
```
{% endstep %}

{% step %}
### Create a Conda Environment

```bash
conda create -n 3dgrut python=3.10 -y
conda activate 3dgrut
```

Your shell prompt should now show `(3dgrut)` at the beginning.
{% endstep %}

{% step %}
### Install PyTorch and Build Tools

Install PyTorch with CUDA support. Even if your system CUDA is 12.8, the `cu121` PyTorch build is compatible:

```bash
pip install torch==2.2.0 torchvision==0.17.0 \
  --index-url https://download.pytorch.org/whl/cu121

pip install ninja cmake
```
{% endstep %}

{% step %}
### Clone the 3DGRUT Repository

If you haven't already:

```bash
cd ~
git clone https://github.com/nv-tlabs/3dgrut.git
cd 3dgrut
```
{% endstep %}

{% step %}
### Download and Install the OptiX SDK

3DGRUT requires the NVIDIA OptiX SDK for ray tracing. The SDK is a self-extracting shell script that can be installed entirely in your home directory — **no sudo required**.

1. Log in to your [NVIDIA Developer account](https://developer.nvidia.com/login).
2. Go to the [OptiX Legacy Downloads page](https://developer.nvidia.com/designworks/optix/downloads/legacy).
3. Download **OptiX SDK 8.0.0 for Linux 64-bit**.
4. Transfer the `.sh` file to your VM (via `scp`, `wget` with a direct link, etc.).

Once the file is on your VM:

```bash
chmod +x ~/3dgrut/NVIDIA-OptiX-SDK-8.0.0-linux64-x86_64.sh

~/3dgrut/NVIDIA-OptiX-SDK-8.0.0-linux64-x86_64.sh \
  --skip-license \
  --prefix=$HOME/optix
```

Set the environment variable so the build system can find it:

```bash
export OptiX_INSTALL_DIR=$HOME/optix
echo 'export OptiX_INSTALL_DIR=$HOME/optix' >> ~/.bashrc
```
{% endstep %}

{% step %}
### Build and Install 3DGRUT

```bash
cd ~/3dgrut
conda activate 3dgrut
pip install -e ".[dev]" --no-build-isolation
```

This will compile all CUDA/C++ extensions in-place. It may take several minutes depending on your GPU and CPU.
{% endstep %}
{% endstepper %}

***

### 4. Prepare Your Scene Data

3DGRUT trains from a set of **posed images** — photos of your scene from multiple angles, along with camera intrinsics and extrinsics. The standard input is **COLMAP**, but NeRF-Synthetic JSON is also supported.

#### Expected Directory Structure (COLMAP)

```
data/
└── my_scene/
    ├── images/          # Your input photos (.jpg or .png)
    │   ├── frame_001.jpg
    │   ├── frame_002.jpg
    │   └── ...
    └── sparse/
        └── 0/
            ├── cameras.bin   # Intrinsics
            ├── images.bin    # Extrinsics (camera poses)
            └── points3D.bin  # Sparse 3D point cloud
```

```bash
cd ~/3dgrut
wget http://storage.googleapis.com/gresearch/refraw360/360_v2.zip
unzip 360_v2.zip -d data/
ls ~/3dgrut/data/
```

***

### 5. Train the Model

Training uses **Hydra** for configuration. There are separate configs for 3DGRT and 3DGUT. The full training loop runs for 30,000 iterations and covers several distinct phases.

#### Understanding the Training Phases

| Phase            | Iterations      | What happens                                                          |
| ---------------- | --------------- | --------------------------------------------------------------------- |
| Warmup           | 0 – 500         | Low learning rate, coarse geometry                                    |
| Densification    | 500 – 15,000    | Clone under-reconstructed Gaussians, split large ones every 100 iters |
| Opacity reset    | 15,000 – 25,000 | Periodically zero out low-opacity Gaussians to prune floaters         |
| Final refinement | 25,000 – 30,000 | Fine-tune colors and covariances, no more densification               |

During densification, the Gaussian count grows from \~50,000 (sparse SfM seed) to several million. Expect GPU memory usage to rise during this phase.

```bash
cd ~/3dgrut
export PATH=$HOME/slang-2024.14.4-linux-x86_64/bin:$PATH
export CUDA_HOME=/usr/local/cuda

python train.py \
  --config-name=apps/colmap_3dgut \
  path=data/room \
  n_iterations=30000 \
  out_dir=outputs/room_3dgut
```

Training will produce:

* `outputs/room_3dgut/<experiment_name>/ckpt_last.pt` — Final checkpoint
* `outputs/room_3dgut/<experiment_name>/ours_7000/ckpt_7000.pt` — Intermediate checkpoint at 7000 iterations
* `outputs/room_3dgut/<experiment_name>/ours_30000/ckpt_30000.pt` — Checkpoint at 30000 iterations
* `outputs/room_3dgut/<experiment_name>/metrics.json` — Evaluation metrics (PSNR, SSIM, LPIPS)
* `outputs/room_3dgut/<experiment_name>/parsed.yaml` — Full resolved config

#### Example Training Results

<figure><img src="../../.gitbook/assets/image (123).png" alt=""><figcaption></figcaption></figure>

### 6.View the Traning Results

The Viser GUI provides a web-based interactive 3D viewer accessible via your browser. This is the best option for remote servers.

**1. Install viser:**

```bash
pip install viser
```

**2. Launch the viewer with your pre-trained checkpoint:**

```bash
cd ~/3dgrut

python train.py \
  --config-name=apps/colmap_3dgut \
  path=data/room \
  with_viser_gui=True \
  test_last=False \
  resume=outputs/room_3dgut/room-1405_080002/ckpt_last.pt
```

You should see:

```
╭────── viser (listening *:8080) ───────╮
│             ╷                         │
│   HTTP      │ http://localhost:8080   │
│   Websocket │ ws://localhost:8080     │
│             ╵                         │
╰───────────────────────────────────────╯
```

**3. On your local machine, set up SSH port forwarding:**

```bash
ssh -L 8080:localhost:8080 ubuntu@<your-vm-ip> -i <private_key>.pem
```

**4. Open in your browser:**

```
http://localhost:8080
```

**5. Navigate the scene:**

On startup you may see a black screen. This is normal. Use your mouse to navigate:

* **Left-click drag** — Rotate
* **Right-click drag** — Pan
* **Scroll wheel** — Zoom

Navigate to the training camera views to see the reconstructed scene.

<figure><img src="../../.gitbook/assets/image (117).png" alt=""><figcaption></figcaption></figure>

***

_Keep an eye on the 3DGRUT GitHub for updates — NVIDIA's team ships improvements regularly. Happy training! 🎉_
