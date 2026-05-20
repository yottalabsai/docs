# Sol-RL on A100: Post-Training Diffusion Models at the Speed of Light

Sol-RL (Speed-of-light RL) is NVIDIA's framework for aligning text-to-image diffusion models with human preferences — significantly faster and cheaper than standard approaches. The core idea: **use NVFP4-quantized rollouts to cheaply explore a massive candidate pool, then train exclusively on the best samples regenerated at full BF16 precision**. The result is up to 4.64× faster convergence with essentially no quality loss, verified across SANA, FLUX.1, and SD3.5-L. This tutorial walks through the paper, the key ideas, and how to run Sol-RL post-training on a Yotta Labs multi-GPU Pod.

RL-based post-training for diffusion models has been picking up serious momentum lately. The basic premise borrows from what made GRPO so effective for LLMs: generate a group of candidate outputs, score them with a reward model, and use the relative advantage between high- and low-reward samples to update the policy. Translated to image generation, this means: for each prompt, sample a batch of images, evaluate them against a human preference signal (ImageReward, CLIPScore, PickScore, HPSv2, etc.), and optimize the diffusion model to produce more of the high-reward kind.

The catch — and it's a real one — is that rollout generation is expensive. On a model like FLUX.1-12B, running BF16 forward passes at scale is brutally slow. The research consistently shows that larger rollout group sizes lead to better alignment (more candidates → sharper advantage signal → cleaner gradient → faster convergence), but scaling rollout group size linearly scales compute. You quickly find yourself in a situation where candidate generation consumes 80% of wall-clock training time, and the policy update itself becomes almost a rounding error. **This is the bottleneck Sol-RL is designed to break.**

### Why FP4 Rollouts Work as a Proxy

The key insight is deceptively simple. In ODE-based diffusion sampling — which covers most modern flow-matching models including SANA, FLUX, and SD3 — the semantic content of a generated image is largely determined by the initial noise seed. Change the denoising path slightly (as FP4 quantization does), and you'll get a different-looking image, but the reward it would receive tends to stay consistent with its BF16 counterpart. The ranking within a group is preserved even when absolute pixel quality degrades.

This means FP4 rollouts can serve as reliable proxy reward rankings. You're not asking them to be perfect images — you're just asking them to tell you which seeds are worth regenerating at full precision. The paper formalizes this in two ways: first, low-precision denoising can be modeled as a bounded perturbation to the BF16 trajectory, so FP4 reward values stay close to their BF16 counterparts; second, as rollout group size N grows, the probability that FP4-identified top/bottom-K seeds match the true BF16 top/bottom-K improves rapidly. In other words, the proxy gets more reliable precisely when you need it most — at large N.

What's notably absent from the paper is any evidence that this ranking consistency holds universally. The authors are careful to scope it to the ODE-style deterministic samplers they use throughout (classifier-free guidance disabled for SANA and SD3.5, guidance embedding fixed at 1.0 for FLUX.1), and to warn that naively using FP4 outputs directly as training targets — without the two-stage decoupling — does degrade quality. The proxy ranking insight only works because you're using FP4 exclusively for selection, not for gradient computation.

### The Two-Stage Pipeline

Each training iteration runs as follows.

**Stage 1 — FP4 Explore.** Sample N=96 initial noise seeds per prompt. Run the NVFP4-quantized model with a reduced step count (fewer denoising steps suffice to rank, not to produce final quality — the paper uses 6 preview steps). Score all 96 images with the reward model. Filter to the top-K and bottom-K seeds (K=24 by default), which form the most contrastive subset and produce the sharpest advantage signal for GRPO.

**Stage 2 — BF16 Train.** Take the 24 selected seeds. Regenerate them from scratch using the full BF16 model with the full denoising step budget. Compute GRPO loss on these high-fidelity regenerations using the proxy rewards from Stage 1. Update the policy weights. Re-quantize to NVFP4 for the next iteration's Stage 1.

The overhead is minimal: Stage 2 runs BF16 on exactly the same number of samples a standard GRPO run would train on anyway. Everything else — the N-K discarded candidates — is replaced with cheap FP4 passes. The paper reports roughly 2% computational overhead from Stage 2 re-quantization, and 2–2.4× rollout phase acceleration from FP4, for a net end-to-end speedup of 1.25× on SANA and 1.62× on FLUX.1. The convergence speedup headline (4.64×) refers to wall-clock time to reach a target reward level — because Sol-RL can afford more iterations in the same time budget, it reaches alignment ceilings that naive BF16 pipelines can't reach without prohibitive cost.

### Results :tada:

Across three base models and four reward metrics, the results are consistent:

**Efficiency (seconds per iteration):**

| Base Model  | Rollout (Naive BF16) | Rollout (Sol-RL) | Speedup | E2E Speedup |
| ----------- | -------------------- | ---------------- | ------- | ----------- |
| FLUX.1      | 184s                 | 79s              | 2.33×   | 1.62×       |
| SD3.5-Large | 451s                 | 187s             | 2.41×   | 1.61×       |
| SANA        | 65s                  | 46s              | 1.41×   | 1.25×       |

**Alignment quality (FLUX.1, ImageReward):**

| Method       | Score      | Δ vs. Base |
| ------------ | ---------- | ---------- |
| DanceGRPO    | 1.4937     | +1.04      |
| FlowGRPO     | 1.5331     | +1.08      |
| AWM          | 1.6693     | +1.21      |
| DiffusionNFT | 1.6707     | +1.22      |
| **Sol-RL**   | **1.7636** | **+1.31**  |

Sol-RL outperforms all baselines on all metrics across all three models while training faster than any of them. The quality gap versus a pure BF16 rollout pipeline (where all N=96 candidates are generated at full precision) is approximately 1% on ImageReward — effectively noise.

***

### Running Sol-RL on a Yotta Labs Pod

The official scripts are configured for **single-node 8-GPU** training, which is the setup validated in the paper. SANA 1.6B is the most accessible starting point — it's the smallest model and the fastest to iterate on. FLUX.1 and SD3.5-L require more VRAM per GPU but run on the same scripts.

#### Prerequisites

* A [Yotta Labs](https://www.yottalabs.ai/) account with sufficient credits
* A [Hugging Face](https://huggingface.co/) account and access token (`HF_TOKEN`)
* Familiarity with multi-GPU JupyterLab environments and `torchrun`

{% stepper %}
{% step %}
### Deploy an 8-GPU Pod

Log in to the [Yotta Labs Console](https://console.yottalabs.ai/). Navigate to **Compute → Pods**, click **Deploy**, and select a **8× GPU** configuration. The official training was done on H100 80GB GPUs; on Yotta Labs, **8× A100** will work for SANA 1.6B with the default config.

| Setting       | Recommended Value                                            |
| ------------- | ------------------------------------------------------------ |
| Template      | `pod-templates-jupyterlab`                                   |
| System Volume | **200 GB** — model weights, checkpoints, reward model caches |
| GPU Count     | **8**                                                        |

Add your Hugging Face token as an environment variable before deploying:

```
HF_TOKEN=<your_huggingface_token>
```

Click **Deploy**, wait for **Running** status, then connect to JupyterLab on port 8888.&#x20;
{% endstep %}

{% step %}
### Clone and Install

Open a terminal via ssh connection and set up the base environment:

```bash
git clone https://github.com/NVlabs/Sana.git
cd Sana
bash ./environment_setup.sh sana
conda activate sana
```

Sol-RL's NVFP4 path (`naive_quant` and `sol_rl` config families) requires `transformer-engine`. Install it with the same Python interpreter that will run `torchrun`:

```bash
python -m pip install --no-build-isolation "transformer-engine[pytorch]"
```

> **Note on `setuptools`:** If you hit a `ModuleNotFoundError: No module named 'pkg_resources'` during setup (caused by setuptools 70+ breaking backwards compatibility), fix it first: `pip install "setuptools<70" --force-reinstall`, then re-run.&#x20;
{% endstep %}

{% step %}
### Download Reward Model Checkpoints

Most reward models download automatically on first use, but **HPSv2** requires manual setup. Create the checkpoint directory and download its weights:

```bash
mkdir -p reward_ckpts
cd reward_ckpts

wget https://huggingface.co/laion/CLIP-ViT-H-14-laion2B-s32B-b79K/resolve/main/open_clip_pytorch_model.bin
wget https://huggingface.co/xswu/HPSv2/resolve/main/HPS_v2.1_compressed.pt

cd ..
```

The other reward models (`clipscore`, `pickscore`, `imagereward`) are downloaded automatically the first time training runs. If you're planning to use only those, you can skip this step.
{% endstep %}

{% step %}
### Run Sol-RL Training on SANA

The repo ships ready-to-run launcher scripts for each model. For SANA, the default entry point is:

```bash
bash train_scripts/sol_rl/run_sana_single_node_8gpu.sh
```

To select a specific config family and reward, set `CONFIG_SPEC` before launching. The config naming pattern is `<model>_<family>_<reward>`. For example, to run the full Sol-RL two-stage pipeline with ImageReward on SANA:

```bash
CONFIG_SPEC=configs/sol_rl/sana.py:sana_sol_rl_imagereward \
bash train_scripts/sol_rl/run_sana_single_node_8gpu.sh
```

If you want to start with something simpler to validate the pipeline first — no NVFP4 required, no `transformer-engine` dependency — use the `diffusionnft` family:

```bash
CONFIG_SPEC=configs/sol_rl/sana.py:sana_diffusionnft_pickscore \
bash train_scripts/sol_rl/run_sana_single_node_8gpu.sh
```

**Config families at a glance:**

| Family          | What it does                                       | NVFP4 needed |
| --------------- | -------------------------------------------------- | ------------ |
| `diffusionnft`  | PEFT-only baseline (24-in-24)                      | No           |
| `naive_scaling` | BF16 brute-force scaling (24-in-96)                | No           |
| `compile`       | BF16 compiled scaling (24-in-96)                   | No           |
| `naive_quant`   | Direct NVFP4 rollout — don't use for real training | Yes          |
| `sol_rl`        | Two-stage decoupled Sol-RL (24-in-96)              | Yes          |

The `diffusionnft` family is the recommended starting point for a first run. It validates that your environment, data loading, and reward scoring all work correctly before you bring in the NVFP4 machinery.&#x20;
{% endstep %}

{% step %}
### Run Inference to Verify Alignment

Once you have a checkpoint (or midway through training), test your aligned model against the base model to see the quality difference:

```bash
python scripts/inference.py \
  --config=configs/sana_config/1024ms/Sana_1600M_img1024.yaml \
  --model_path=output/<your_checkpoint_dir>/checkpoints/latest.pth \
  --txt_file=asset/samples_mini.txt
```

To quantify the improvement with reward scores, use the ImageReward evaluation script:

```bash
bash scripts/bash_run_inference_metric_imagereward.sh \
  configs/sana_config/1024ms/Sana_1600M_img1024.yaml \
  output/<your_checkpoint_dir>/checkpoints/latest.pth
```

The alignment improvement is most visible on compositionally complex prompts — spatial relationships, counting, attribute binding. The RL-aligned model handles "a red cube on top of a blue sphere to the left of a green cylinder" noticeably better than the base model, because the reward signal specifically pushes the policy toward prompt fidelity.&#x20;
{% endstep %}
{% endstepper %}

***

### Troubleshooting

**`transformer-engine` install fails.** This package is tied tightly to your CUDA and PyTorch versions. Run `python -c "import torch; print(torch.__version__, torch.version.cuda)"` first, then check the transformer-engine release page for a matching wheel. On Blackwell (RTX 5090 / H100), CUDA 12.4+ and PyTorch 2.4+ are required.

**`run_sana_single_node_8gpu.sh` exits immediately with no output.** Usually means `torchrun` can't find all 8 GPUs. Check `nvidia-smi` — all 8 should be visible with no processes holding them. If another process is using a GPU, kill it first.

**Reward scores don't improve after 100+ iterations.** First check that the reward model is actually scoring correctly by looking at per-sample scores in the training logs. If scores are near-constant, the reward model may have failed to load (common if `reward_ckpts/` is missing for HPSv2). Switch to `pickscore` or `imagereward` which auto-download and are more robust to environment issues.

**OOM during the BF16 Stage 2 regeneration.** Reduce `train_group_size` (K) from 24 to 16 in your config. This reduces the BF16 batch size with minimal impact on gradient quality, since you're still selecting from the same N=96 FP4 candidates.

**Want to run on FLUX.1 or SD3.5-L?** The launchers are identical in structure, just swap the script:

```bash
# FLUX.1
CONFIG_SPEC=configs/sol_rl/flux1.py:flux1_sol_rl_imagereward \
bash train_scripts/sol_rl/run_flux1_single_node_8gpu.sh

# SD3.5-Large
CONFIG_SPEC=configs/sol_rl/sd3.py:sd3_sol_rl_imagereward \
bash train_scripts/sol_rl/run_sd3_single_node_8gpu.sh
```

FLUX.1 shows the largest absolute efficiency gains (2.33× rollout speedup), making it the most rewarding target if you have the VRAM budget.

***

_Resources:_ [_Paper (arXiv:2604.06916)_](https://arxiv.org/abs/2604.06916) _·_ [_GitHub_](https://github.com/NVlabs/Sana) _·_ [_Project Page_](https://nvlabs.github.io/Sana/Sol-RL/) _·_ [_Official Sol-RL Docs_](https://nvlabs.github.io/Sana/docs/sol_rl/)
