# SkyRL for LLM-as-a-Judge Training

### What is  LLM-as-a-Judge About?

Imagine teaching a student mathematics. The traditional approach is to show them correct answers and have them memorize it. But a smarter method is: let them solve problems on their own, then have a "judge teacher" evaluate whether their answers are good or not. Reward good work and correct mistakes. **That's exactly what we're doing here!**

We're using the SkyRL framework to train a small language model `Qwen2.5-1.5B` to solve math problems (GSM8K dataset). `GPT-4o-mini` acts as the "judge teacher," evaluating the quality of the model's answers.

{% hint style="info" %}
_**The GSM8K (Grade School Math 8K) dataset** is a collection of 8.5K high-quality, linguistically diverse grade school math word problems. This dataset was created to support the task of question answering on basic mathematical problems that require multi-step reasoning._&#x20;

_It looks like:_

```json
{
"question": "Natalia sold clips to 48 of her friends in April, and then she sold half as many clips in May. How many clips did Natalia sell altogether in April and May?",
"answer": "Natalia sold 48/2 = <<48/2=24>>24 clips in May.\nNatalia sold 48+24 = <<48+24=72>>72 clips altogether in April and May.\n#### 72"
}
```
{% endhint %}

***

### Step-by-Step Guide

{% stepper %}
{% step %}
### Environment Check&#x20;

```python
# Start a pod and connect to 8888 jupyterlab port.Start a new notebook.
# We need to confirm the SkyRL project code is downloaded
# Use git clone https://github.com/NovaSky-AI/SkyRL.git
import os

# Check current directory structure
print("Current working directory:", os.getcwd())
print("\nDirectory structure:")
!ls -la

# Check if SkyRL exists
if os.path.exists('SkyRL'):
    print("\n✓ SkyRL directory exists")
    !ls -la SkyRL/
else:
    print("\n✗ SkyRL directory not found, need to clone first")
```

Checks if the `SkyRL` folder exists in the current directory
{% endstep %}

{% step %}
### &#x20;Locate Key Files

```python
# Navigate to SkyRL's training code directory
# Check if the three important files we need are present
import os

# Change to skyrl-train directory
os.chdir('/workspace/SkyRL/skyrl-train')
print("Current directory:", os.getcwd())

# Check key files
print("\nChecking key files:")
files_to_check = [
    'examples/llm_as_a_judge/gsm8k_dataset_judge.py',
    'examples/llm_as_a_judge/main_llm_judge.py',
    'examples/llm_as_a_judge/run_llm_judge.sh',
]

for file in files_to_check:
    if os.path.exists(file):
        print(f"✓ {file}")
    else:
        print(f"✗ {file}")
```

**Files to check**:

* `gsm8k_dataset_judge.py` - Data preparation script, organizing math problems into training format
* `main_llm_judge.py` - Main training program
* `run_llm_judge.sh` - Quick launch script
{% endstep %}

{% step %}
### Prepare Training Data&#x20;

```python
# Organize raw math problem data into a format the model can "digest"
# Generate training and validation sets
import os

# Set data output directory
DATA_DIR = os.path.expanduser("~/data/gsm8k_llm_judge")
print(f"Data will be saved to: {DATA_DIR}")

# Run dataset preparation script
command = f"uv run examples/llm_as_a_judge/gsm8k_dataset_judge.py --output_dir {DATA_DIR}"
print(f"\nExecuting command:\n{command}\n")
!{command}

# Verify dataset
print("\nVerifying dataset:")
!ls -lh {DATA_DIR}
```

**Output files**:

* `train.parquet` - Training data (for model practice)
* `validation.parquet` - Validation data (to check how well the model learned)

**Data saved to**: `~/data/gsm8k_llm_judge/` directory
{% endstep %}

{% step %}
### Install System Dependencies&#x20;

```bash
# Install libnuma-dev
# This is a low-level library that helps programs better utilize multi-core CPUs
!sudo apt-get update
!sudo apt-get install -y libnuma-dev
```

NUMA (Non-Uniform Memory Access) library optimizes memory access performance
{% endstep %}

{% step %}
### Configure API Keys&#x20;

```python
# Create .env.llm_judge configuration file
# Contains:
# - OpenAI API Key (for calling GPT-4o-mini as the judge)
# - Wandb config (experiment tracking tool, disabled here)
# Create .env.llm_judge file
env_content = """OPENAI_API_KEY=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
WANDB_API_KEY=dummy_key_not_used
WANDB_MODE=disabled
"""

with open('.env.llm_judge', 'w') as f:
    f.write(env_content)

print("✓ .env.llm_judge created")

# Verify file content
print("\nFile content:")
!cat .env.llm_judge
```

:exclamation:Replace with your own real OpenAI API Key!
{% endstep %}

{% step %}
### Start Training!&#x20;

Here comes the coolest part.

```python
import os

# ==================== Configuration Section ====================
# Set the directory where GSM8K dataset will be stored
DATA_DIR = os.path.expanduser("~/data/gsm8k_llm_judge")
# Number of GPUs to use for training 
NUM_GPUS = 1 
#Adjust this according to the actual number of GPUs you use!!
#If you check on the original doc skyrl team provided, you'll find the default is 4.


# ==================== Change Directory ====================
# Navigate to the skyrl-train module directory
os.chdir('/workspace/SkyRL/skyrl-train')
print(f"✓ Current directory: {os.getcwd()}\n")
print("=" * 80)
print("Starting Training (1 GPU Config - CPU Offload Disabled)")
print("=" * 80)

# CRITICAL: Disable CPU offload for single GPU to avoid CPU-GPU transfer bottlenecks
command = f"""
uv run --isolated --extra vllm --env-file .env.llm_judge -m examples.llm_as_a_judge.main_llm_judge \
  data.train_data="['{DATA_DIR}/train.parquet']" \              # Training dataset path
  data.val_data="['{DATA_DIR}/validation.parquet']" \           # Validation dataset path
  trainer.algorithm.advantage_estimator="grpo" \                # Use GRPO (Group Relative Policy Optimization)
  trainer.policy.model.path="Qwen/Qwen2.5-1.5B-Instruct" \      # Base model: Qwen 1.5B Instruct
  trainer.epochs=20 \                                            # Train for 20 epochs
  trainer.train_batch_size=8 \                                   # Process 8 samples per batch
  trainer.policy_mini_batch_size=8 \                             # Mini-batch size for policy updates
  trainer.placement.colocate_all=true \                          # Place all models on the same GPU
  trainer.strategy=fsdp2 \                                       # Use FSDP2 (Fully Sharded Data Parallel v2)
  trainer.placement.policy_num_gpus_per_node={NUM_GPUS} \        # Policy model GPU allocation
  trainer.placement.ref_num_gpus_per_node={NUM_GPUS} \           # Reference model GPU allocation
  trainer.placement.critic_num_gpus_per_node={NUM_GPUS} \        # Critic model GPU allocation
  trainer.policy.fsdp_config.cpu_offload=false \                 # Disable CPU offload for policy model
  trainer.ref.fsdp_config.cpu_offload=false \                    # Disable CPU offload for reference model
  trainer.critic.fsdp_config.cpu_offload=false \                 # Disable CPU offload for critic model
  generator.num_inference_engines=1 \                            # Use 1 inference engine
  generator.inference_engine_tensor_parallel_size=1 \            # No tensor parallelism
  generator.backend=vllm \                                       # Use vLLM as inference backend
  generator.n_samples_per_prompt=5 \                             # Generate 5 candidate answers per question
  environment.env_class=llm_as_a_judge \                         # Use LLM-as-a-Judge environment
  environment.skyrl_gym.llm_as_a_judge.model="gpt-4o-mini"      # GPT-4o-mini as the judge model
"""

!{command}
```
{% endstep %}
{% endstepper %}

***

### &#x20;How Does the Training Process Work?

#### 1️⃣ **Generation Phase**

The student model (Qwen2.5-1.5B) sees a math problem and attempts to generate 5 different answers

#### 2️⃣ **Evaluation Phase**

The judge model (GPT-4o-mini) reviews these 5 answers and scores each one:

* ✅ Correct answer, clear reasoning → High reward
* ⚠️ Correct answer, but messy process → Medium reward
* ❌ Wrong answer → Low score or penalty

#### 3️⃣ **Learning Phase**

Based on the judge's scores, the student model adjusts its "problem-solving strategy":

* High-scoring approaches → Use more often
* Low-scoring approaches → Use less often

#### 4️⃣ **Repeat Cycle**

Repeat for 20 rounds (20 epochs), the student model gradually learns problem-solving skills

{% hint style="info" %}
Need more information? Check these:&#x20;

* [SkyRL GitHub Repository](https://github.com/skyworkai/skyrl)
* [GSM8K Dataset Paper](https://arxiv.org/abs/2110.14168)
{% endhint %}

Have fun building!
