---
hidden: true
---

# Reinforced Learning with Miles

### Prerequisites

Access to a YottaLabs RTX 5090 GPU, ideally.

Start a pod with our official template and enter Jupyter lab.

***

Run the cell below to see what we're working with:

```python
# Let's see what awesome hardware we have! 🖥
!nvidia-smi

# Check Python version
import sys
print(f"\n🐍 Python version: {sys.version}")

# Check current working directory
import os
print(f"\n📁 Current directory: {os.getcwd()}")
```

<figure><img src="../../.gitbook/assets/image (139).png" alt="" width="563"><figcaption></figcaption></figure>

***

{% stepper %}
{% step %}
### &#x20;Step 1: Environment Setup

```python
# Set up our working directory
# YottaLabs users: we're using /workspace/ as the base directory
WORK_DIR = "/workspace/miles_workspace"
os.makedirs(WORK_DIR, exist_ok=True)
os.chdir(WORK_DIR)

print(f"✅ Working directory set to: {WORK_DIR}")
```

{% hint style="info" %}
The original miles repository quickstart provides a `build_conda.sh` script, but we'll adapt it for our JupyterLab environment.
{% endhint %}

Let's grab the miles framework from GitHub. This is where all the magic happens.

```python
# Clone miles repository if it doesn't exist
if not os.path.exists(f"{WORK_DIR}/miles"):
    !git clone https://github.com/radixark/miles.git
    print("✅ Miles repository cloned successfully!")
else:
    print("✅ Miles repository already exists!")
    # Let's make sure we have the latest version
    %cd {WORK_DIR}/miles
    !git pull
    %cd {WORK_DIR}
```

Now let's install miles and its dependencies. This might take a few minutes.

```python
%cd {WORK_DIR}/miles

# Install miles in editable mode
%pip install -e . --no-deps

print("\n✅ Miles installation complete!")
```

Miles uses Megatron-LM as one of its training backends.&#x20;

```python
%cd {WORK_DIR}

if not os.path.exists(f"{WORK_DIR}/Megatron-LM"):
    %git clone https://github.com/NVIDIA/Megatron-LM.git
    print("✅ Megatron-LM cloned successfully!")
else:
    print("✅ Megatron-LM already exists!")

# Add Megatron to Python path
import sys
megatron_path = f"{WORK_DIR}/Megatron-LM"
if megatron_path not in sys.path:
    sys.path.append(megatron_path)
    
# Also set it as an environment variable for subprocess calls
os.environ['PYTHONPATH'] = f"{megatron_path}:{os.environ.get('PYTHONPATH', '')}"

print(f"✅ Megatron-LM added to Python path")
```
{% endstep %}

{% step %}
### Prepare Models and Datasets

We'll be working with:

* **Model:** GLM-Z1-9B (a 9 billion parameter language model)
* **Training Data:** dapo-math-17k (17,000 math problems)
* **Evaluation Data:** aime-2024 (American Invitational Mathematics Examination)

These downloads can take a while depending on your connection.&#x20;

Let's organize our downloads neatly.&#x20;

```python
# Create directories for models and data
MODEL_DIR = f"{WORK_DIR}/models"
DATA_DIR = f"{WORK_DIR}/datasets"

os.makedirs(MODEL_DIR, exist_ok=True)
os.makedirs(DATA_DIR, exist_ok=True)

print(f"✅ Models will be saved to: {MODEL_DIR}")
print(f"✅ Datasets will be saved to: {DATA_DIR}")
```

We're using `huggingface-cli` to download the GLM-Z1-9B model. This is a fantastic 9B parameter model perfect for learning.

```python

import sys
import subprocess

subprocess.check_call([sys.executable, "-m", "pip", "install", "-U", "huggingface_hub"])

from huggingface_hub import snapshot_download

MODEL_NAME = "GLM-Z1-9B-0414"
MODEL_PATH = f"{MODEL_DIR}/{MODEL_NAME}"

if not os.path.exists(MODEL_PATH):
    print(f"🚀 Downloading {MODEL_NAME}... This might take 10-20 minutes.")
    snapshot_download(
        repo_id="zai-org/GLM-Z1-9B-0414",
        local_dir=MODEL_PATH,
        local_dir_use_symlinks=False
    )
    print(f"\n✅ Model downloaded to: {MODEL_PATH}")
else:
    print(f"✅ Model already exists at: {MODEL_PATH}")
```

<figure><img src="../../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

The dapo-math-17k dataset contains 17,000 math problems. Great for training our model to think mathematically.

```python
TRAIN_DATA_PATH = f"{DATA_DIR}/dapo-math-17k"

if not os.path.exists(TRAIN_DATA_PATH):
    print("📚 Downloading training dataset...")
    !huggingface-cli download --repo-type dataset zhuzilin/dapo-math-17k --local-dir {TRAIN_DATA_PATH}
    print(f"\n✅ Training data downloaded to: {TRAIN_DATA_PATH}")
else:
    print(f"✅ Training data already exists at: {TRAIN_DATA_PATH}")
```

<figure><img src="../../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

The AIME-2024 dataset will help us evaluate how well our model is learning. Think of it as the final exam.

```python
EVAL_DATA_PATH = f"{DATA_DIR}/aime-2024"

if not os.path.exists(EVAL_DATA_PATH):
    print("📊 Downloading evaluation dataset...")
    !huggingface-cli download --repo-type dataset zhuzilin/aime-2024 --local-dir {EVAL_DATA_PATH}
    print(f"\n✅ Evaluation data downloaded to: {EVAL_DATA_PATH}")
else:
    print(f"✅ Evaluation data already exists at: {EVAL_DATA_PATH}")
```

<figure><img src="../../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>
{% endstep %}

{% step %}
### Model Weight Conversion

Miles uses Megatron for training, which requires a specific weight format. We need to convert our Hugging Face model to Megatron's `torch_dist` format.

Think of this as translating a book from one language to another - same content, different format.

First, we need to load the model's configuration. Miles provides ready-made configs for popular models.

```python
# Let's peek at what the GLM4-9B config looks like
config_file = f"{WORK_DIR}/miles/scripts/models/glm4-9B.sh"

print("📋 Model configuration parameters:")
print("=" * 50)
with open(config_file, 'r') as f:
    content = f.read()
    print(content)
print("=" * 50)
```

Now let's do the actual conversion! This process reads the Hugging Face weights and reorganizes them into Megatron's format.

```python
%cd {WORK_DIR}/miles

# Output path for converted weights
CONVERTED_MODEL_PATH = f"{MODEL_PATH}_torch_dist"

if not os.path.exists(CONVERTED_MODEL_PATH):
    print("🔄 Converting model weights to Megatron format...")
    print("This will take a while - perfect time to learn about what's happening!\n")
    print("📖 During conversion, we're:")
    print("   1. Reading the Hugging Face model structure")
    print("   2. Redistributing weights for tensor parallelism")
    print("   3. Saving in Megatron's checkpoint format\n")
    
    # Source the model config and run conversion
    !bash -c "source scripts/models/glm4-9B.sh && \
             PYTHONPATH={WORK_DIR}/Megatron-LM python tools/convert_hf_to_torch_dist.py \
             --hf-checkpoint {MODEL_PATH} \
             --save {CONVERTED_MODEL_PATH} \
             --num-layers 40 \
             --hidden-size 4096 \
             --num-attention-heads 32 \
             --group-query-attention \
             --num-query-groups 2 \
             --ffn-hidden-size 13696 \
             --seq-length 131072 \
             --max-position-embeddings 131072 \
             --rotary-base 5000000 \
             --rotary-percent 0.5 \
             --swiglu \
             --tokenizer-type PretrainedFromHF \
             --no-bias-swiglu-fusion \
             --layernorm-type RMSNorm \
             --untie-embeddings-and-output-weights \
             --disable-bias-linear \
             --normalization RMSNorm"
    
    print(f"\n✅ Conversion complete! Saved to: {CONVERTED_MODEL_PATH}")
else:
    print(f"✅ Converted model already exists at: {CONVERTED_MODEL_PATH}")
```


{% endstep %}

{% step %}
### Understanding Training Parameters

Before we start training, let's take a moment to understand what's going on under the hood. Trust me, this will make everything click.

Miles training follows a reinforcement learning loop:

```
📊 Data Sampling (Rollout) → 🎓 Weight Update (Training) → 🔄 Repeat
```

Let me break down the key parameters that control this loop:

**Phase 1: Data Sampling (Rollout)**

* **`rollout-batch-size`**: How many prompts we sample each round
  * Example: 16 prompts
* **`n-samples-per-prompt`**: How many responses we generate per prompt
  * Example: 8 responses per prompt
* **Total samples generated** = 16 × 8 = 128 samples

**Phase 2: Model Training**

* **`global-batch-size`**: Samples needed for one parameter update
  * Example: 128 samples
* **`num-steps-per-rollout`**: How many updates to make with current data
  * Example: 1 update (on-policy learning)
* **Total samples consumed** = 128 × 1 = 128 samples

{% hint style="info" %}
**Golden Rule 🌟**

Samples Generated = Samples Consumed

(rollout-batch-size × n-samples-per-prompt) = (global-batch-size × num-steps-per-rollout)
{% endhint %}

This ensures we use exactly the data we generate - no waste, no shortfall.

```python
# Let's visualize this with a quick calculation helper
def validate_training_params(rollout_batch, n_samples, global_batch, num_steps):
    generated = rollout_batch * n_samples
    consumed = global_batch * num_steps
    
    print("📊 Training Loop Balance Check")
    print("=" * 50)
    print(f"🎲 Samples Generated: {rollout_batch} × {n_samples} = {generated}")
    print(f"🎓 Samples Consumed:  {global_batch} × {num_steps} = {consumed}")
    print("=" * 50)
    
    if generated == consumed:
        print("✅ Perfect balance! Your parameters are correct!")
        return True
    else:
        print(f"❌ Imbalance detected! Difference: {abs(generated - consumed)}")
        print("💡 Tip: Adjust your parameters to match the golden rule.")
        return False

# Example configuration
validate_training_params(
    rollout_batch=16,
    n_samples=8,
    global_batch=128,
    num_steps=1
)
```


{% endstep %}

{% step %}
### Prepare Your Training Script

Alright, time to put it all together.

```python
# Let's create our training configuration
import json

# Base paths
TRAINING_CONFIG = {
    # Model paths
    "hf_checkpoint": MODEL_PATH,
    "ref_load": CONVERTED_MODEL_PATH,
    "load": f"{MODEL_PATH}_miles_checkpoint",
    "save": f"{MODEL_PATH}_miles_checkpoint",
    "save_interval": 20,
    
    # Data paths
    "prompt_data": f"{TRAIN_DATA_PATH}/dapo-math-17k.jsonl",
    "eval_prompt_data": f"{EVAL_DATA_PATH}/aime-2024.jsonl",
    
    # Training loop parameters
    "num_rollout": 100,  # Reduced for quick testing
    "rollout_batch_size": 16,
    "n_samples_per_prompt": 8,
    "num_steps_per_rollout": 1,
    "global_batch_size": 128,
    
    # Model configuration (GLM4-9B)
    "num_layers": 40,
    "hidden_size": 4096,
    "num_attention_heads": 32,
    "num_query_groups": 2,
    "ffn_hidden_size": 13696,
    "seq_length": 131072,
    
    # Parallelism (adjust based on your GPU count)
    "tensor_model_parallel_size": 2,
    "pipeline_model_parallel_size": 1,
    "context_parallel_size": 2,
    
    # Performance
    "max_tokens_per_gpu": 4608,
    "use_dynamic_batch_size": True,
    
    # GRPO algorithm
    "advantage_estimator": "grpo",
    "kl_loss_coef": 0.0,
    
    # Optimizer
    "optimizer": "adam",
    "lr": 1e-6,
    "weight_decay": 0.1,
}

# Save config for reference
config_path = f"{WORK_DIR}/training_config.json"
with open(config_path, 'w') as f:
    json.dump(TRAINING_CONFIG, f, indent=2)

print("✅ Training configuration created!")
print(f"📄 Saved to: {config_path}")
print("\n📋 Quick Summary:")
print(f"   • Training for {TRAINING_CONFIG['num_rollout']} rollouts")
print(f"   • {TRAINING_CONFIG['rollout_batch_size']} prompts per rollout")
print(f"   • {TRAINING_CONFIG['n_samples_per_prompt']} samples per prompt")
print(f"   • Total samples per rollout: {TRAINING_CONFIG['rollout_batch_size'] * TRAINING_CONFIG['n_samples_per_prompt']}")
```

<figure><img src="../../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>
{% endstep %}

{% step %}
### Start Ray and Launch Training

Now for the exciting part - let's actually start training.

Miles uses Ray for distributed computing. We'll need to:

1. Start a Ray cluster
2. Submit our training job
3. Monitor progress

Ray is pretty awesome - it handles all the distributed computing magic for us✨Let's start ray.

```python
# Check if Ray is already running
!ray status || ray start --head --num-gpus=8 --disable-usage-stats

print("\n✅ Ray cluster is ready!")
print("\n💡 Tip: You can view the Ray dashboard in your browser")
print("   Default URL: http://localhost:8265")
```

Let's build our training command. I'll show you each part so you understand what's happening!

```python
%cd {WORK_DIR}/miles

# Build the training command
# We're using a background process so you can monitor it in real-time

training_script = f"""
export PYTHONPATH={WORK_DIR}/Megatron-LM:$PYTHONPATH

python3 train.py \\
    --actor-num-nodes 1 \\
    --actor-num-gpus-per-node 8 \\
    --rollout-num-gpus 8 \\
    --rollout-num-gpus-per-engine 2 \\
    \\
    --hf-checkpoint {TRAINING_CONFIG['hf_checkpoint']} \\
    --ref-load {TRAINING_CONFIG['ref_load']} \\
    --load {TRAINING_CONFIG['load']} \\
    --save {TRAINING_CONFIG['save']} \\
    --save-interval {TRAINING_CONFIG['save_interval']} \\
    \\
    --prompt-data {TRAINING_CONFIG['prompt_data']} \\
    --input-key prompt \\
    --label-key label \\
    --apply-chat-template \\
    --rollout-shuffle \\
    \\
    --rm-type deepscaler \\
    \\
    --num-rollout {TRAINING_CONFIG['num_rollout']} \\
    --rollout-batch-size {TRAINING_CONFIG['rollout_batch_size']} \\
    --n-samples-per-prompt {TRAINING_CONFIG['n_samples_per_prompt']} \\
    --num-steps-per-rollout {TRAINING_CONFIG['num_steps_per_rollout']} \\
    --global-batch-size {TRAINING_CONFIG['global_batch_size']} \\
    \\
    --rollout-max-response-len 8192 \\
    --rollout-temperature 1 \\
    --balance-data \\
    \\
    --eval-interval 5 \\
    --eval-prompt-data aime {TRAINING_CONFIG['eval_prompt_data']} \\
    --n-samples-per-eval-prompt 16 \\
    --eval-max-response-len 16384 \\
    --eval-top-p 1 \\
    \\
    --tensor-model-parallel-size {TRAINING_CONFIG['tensor_model_parallel_size']} \\
    --sequence-parallel \\
    --pipeline-model-parallel-size {TRAINING_CONFIG['pipeline_model_parallel_size']} \\
    --context-parallel-size {TRAINING_CONFIG['context_parallel_size']} \\
    --recompute-granularity full \\
    --recompute-method uniform \\
    --recompute-num-layers 1 \\
    --use-dynamic-batch-size \\
    --max-tokens-per-gpu {TRAINING_CONFIG['max_tokens_per_gpu']} \\
    \\
    --advantage-estimator {TRAINING_CONFIG['advantage_estimator']} \\
    --use-kl-loss \\
    --kl-loss-coef {TRAINING_CONFIG['kl_loss_coef']} \\
    --kl-loss-type low_var_kl \\
    --entropy-coef 0.0 \\
    --eps-clip 0.2 \\
    --eps-clip-high 0.28 \\
    \\
    --optimizer {TRAINING_CONFIG['optimizer']} \\
    --lr {TRAINING_CONFIG['lr']} \\
    --lr-decay-style constant \\
    --weight-decay {TRAINING_CONFIG['weight_decay']} \\
    --adam-beta1 0.9 \\
    --adam-beta2 0.98 \\
    \\
    --num-layers {TRAINING_CONFIG['num_layers']} \\
    --hidden-size {TRAINING_CONFIG['hidden_size']} \\
    --num-attention-heads {TRAINING_CONFIG['num_attention_heads']} \\
    --group-query-attention \\
    --num-query-groups {TRAINING_CONFIG['num_query_groups']} \\
    --ffn-hidden-size {TRAINING_CONFIG['ffn_hidden_size']} \\
    --seq-length {TRAINING_CONFIG['seq_length']} \\
    --max-position-embeddings {TRAINING_CONFIG['seq_length']} \\
    --rotary-base 5000000 \\
    --rotary-percent 0.5 \\
    --swiglu \\
    --tokenizer-type PretrainedFromHF \\
    --no-bias-swiglu-fusion \\
    --layernorm-type RMSNorm \\
    --untie-embeddings-and-output-weights \\
    --disable-bias-linear \\
    --normalization RMSNorm
"""

# Save the script
script_path = f"{WORK_DIR}/run_training.sh"
with open(script_path, 'w') as f:
    f.write(training_script)

!chmod +x {script_path}

print("✅ Training script created!")
print(f"📄 Saved to: {script_path}")
print("\n🎯 Ready to launch training!")
```

This is it - the moment we've been building up to.

Training will run in the background. You can monitor progress in the next cell.

```python
# Launch training
print("🚀 Launching training job...")
print("This will take several hours depending on your hardware.\n")

!bash {script_path} > {WORK_DIR}/training.log 2>&1 &

print("✅ Training started!")
print(f" Log file: {WORK_DIR}/training.log")
print("\n Tips for monitoring:")
print("   • Check the log file for detailed progress")
print("   • Use Ray dashboard: http://localhost:8265")
print("   • Run the monitoring cell below for real-time updates")
```

Let's keep an eye on how things are going! This cell will show you the latest updates from the training log.

```python
# Display the last 50 lines of the training log
import time

log_file = f"{WORK_DIR}/training.log"

if os.path.exists(log_file):
    print("📊 Latest Training Updates")
    print("=" * 70)
    !tail -50 {log_file}
    print("=" * 70)
    print(f"\n⏰ Last checked: {time.strftime('%Y-%m-%d %H:%M:%S')}")
    print("\n💡 Tip: Re-run this cell anytime to see fresh updates!")
else:
    print("⏳ Log file not created yet. Training is starting up...")
    print("Wait a minute and try again!")
```
{% endstep %}

{% step %}
### Understanding What's Happening

While training runs, let me explain what's happening behind the scenes.&#x20;

#### The GRPO Algorithm

Miles uses GRPO (Group Relative Policy Optimization) by default. Here's what's happening in each iteration:

✅**Sampling Phase**&#x20;

* Model generates multiple responses for each prompt
* Each response gets a reward score from the reward model
* We collect both good and bad responses

✅**Advantage Calculation**&#x20;

* Compare each response's reward within its group
* Responses better than the group average get positive advantages
* Poor responses get negative advantages

✅**Policy Update**&#x20;

* Update the model to increase probability of high-advantage responses
* Decrease probability of low-advantage responses
* Use PPO-style clipping to prevent drastic changes

✅**Evaluation**&#x20;

* Every few rollouts, test on the AIME dataset
* Track progress and save checkpoints

In the logs, you'll see these key metrics:

* **`loss`**: Training loss (should decrease over time)
* **`reward_mean`**: Average reward score (should increase)
* **`kl_divergence`**: How much model changes (want controlled change)
* **`eval_accuracy`**: Performance on test set (the ultimate goal!)
{% endstep %}
{% endstepper %}
