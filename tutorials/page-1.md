---
icon: rainbow
---

# Page 1

This guide shows how to run a **working SkyRL training job on YottaLabs** virtual machine using:

* **1 H200 GPU**
* **Docker**
* **SkyRL**
* **GSM8K**
* **Qwen/Qwen2.5-1.5B-Instruct**
* **GRPO + FSDP2**
* **vLLM backend**

***

{% stepper %}
{% step %}
### Connect to your YottaLabs instance

Start a virtual machine on our platform. See our [Virtual Machine Guide](https://docs.yottalabs.ai/yotta-labs/products/virtual-machines/launching-a-virtual-machine) for step-to-step tutorial.

From your local machine, SSH into the YottaLabs host, for example:

```bash
ssh root@instance-2037157542617944064.yottadeos.com -i private_key.pem
```

Once connected, you should be on the remote machine as `root`.
{% endstep %}

{% step %}
### Start the SkyRL Docker container

Run the official training container with GPU access enabled:

```bash
docker run -it \
  --runtime=nvidia \
  --gpus all \
  --shm-size=16g \
  --name skyrl \
  novaskyai/skyrl-train-ray-2.51.1-py3.12-cu12.8 \
  /bin/bash
```

This container already includes the right CUDA-compatible environment for SkyRL training. We also give Docker a larger shared memory segment with:

```bash
--shm-size=16g
```

That helps Ray and PyTorch behave better during training.

Once the container starts, you should land inside a shell that looks roughly like this:

```
(base) ray@...:~$
```
{% endstep %}

{% step %}
### Clone the SkyRL repository

Inside the container, run:

```bash
git clone https://github.com/novasky-ai/SkyRL.git
cd SkyRL
```

At this point you should be in:

<pre class="language-bash"><code class="lang-bash"><strong>/home/ray/SkyRL
</strong></code></pre>
{% endstep %}

{% step %}
### Prepare the GSM8K dataset

Generate the parquet files SkyRL expects for training and validation:

```bash
uv run --isolated examples/train/gsm8k/gsm8k_dataset.py --output_dir $HOME/data/gsm8k
```

This command downloads and preprocesses GSM8K into parquet format.

When it finishes, verify that the dataset files exist:

```bash
ls -lh $HOME/data/gsm8k
```

You should see files such as:

```
train.parquet
validation.parquet
```

That confirms the data preparation step is done.
{% endstep %}

{% step %}
### Export your W\&B API key

SkyRL logs this run to Weights & Biases, so set your API key before training:

```bash
export WANDB_API_KEY=your_wandb_api_key_here
```

Use your real key in place of the placeholder above.

If you want to confirm the variable exists without printing the key itself, you can run:

```bash
echo $WANDB_API_KEY | wc -c
```

A nonzero result means the variable is present.
{% endstep %}

{% step %}
### Launch training

Here is the exact single-GPU training command.

Use the **one-line version** for the least error-prone performance in shell environments:

```bash
uv run --isolated --extra fsdp -m skyrl.train.entrypoints.main_base data.train_data="['$HOME/data/gsm8k/train.parquet']" data.val_data="['$HOME/data/gsm8k/validation.parquet']" trainer.algorithm.advantage_estimator="grpo" trainer.policy.model.path="Qwen/Qwen2.5-1.5B-Instruct" trainer.strategy=fsdp2 trainer.placement.colocate_all=true trainer.placement.policy_num_gpus_per_node=1 trainer.eval_batch_size=1024 trainer.eval_before_train=true trainer.eval_interval=5 trainer.ckpt_interval=10 generator.inference_engine.backend=vllm generator.inference_engine.num_engines=1 generator.inference_engine.tensor_parallel_size=1 generator.inference_engine.weight_sync_backend=nccl environment.env_class=gsm8k
```

Let us briefly decode the important parts so the command is not just a magic spell.

### Data

```
data.train_data="['$HOME/data/gsm8k/train.parquet']"
data.val_data="['$HOME/data/gsm8k/validation.parquet']"
```

These point SkyRL to the GSM8K parquet files you just generated.

### Algorithm

```
trainer.algorithm.advantage_estimator="grpo"
```

This tells SkyRL to train with **GRPO**.

### Base model

```
trainer.policy.model.path="Qwen/Qwen2.5-1.5B-Instruct"
```

This is the policy model being trained.

### Training strategy

```
trainer.strategy=fsdp2
trainer.placement.colocate_all=true
trainer.placement.policy_num_gpus_per_node=1
```

This configures a **single-GPU FSDP2 run** and keeps the components colocated appropriately for a one-GPU machine.

### Evaluation

```
trainer.eval_batch_size=1024
trainer.eval_before_train=true
trainer.eval_interval=5
trainer.ckpt_interval=10
```

This tells SkyRL to:

* run evaluation before training starts
* evaluate every 5 steps
* save checkpoints every 10 steps

### Inference engine

```
generator.inference_engine.backend=vllm
generator.inference_engine.num_engines=1
generator.inference_engine.tensor_parallel_size=1
generator.inference_engine.weight_sync_backend=nccl
```

This uses **vLLM** for rollout generation with a **single engine** and **tensor parallel size 1**, which matches a single-GPU setup.

### Environment

```
environment.env_class=gsm8k
```

This tells SkyRL to use the GSM8K task environment for reward computation and evaluation.
{% endstep %}

{% step %}
### What a healthy startup looks like

After launching training, you should see logs that indicate the system is booting correctly.

Here are the kinds of messages you want to see:

```
Started a local Ray instance.
Infrastructure logs will be written to: /tmp/skyrl-logs/...
Synced registries to ray actor
InferenceEngineClient initialized with 1 engines.
Total steps: 7
Validation set size: 2
init policy/ref/critic models done
No checkpoint found, starting training from scratch
Started: 'eval'
```

These lines tell you:

* Ray started
* SkyRL initialized its worker stack
* the inference engine came up
* the dataset loaded
* the trainer is entering evaluation and then training
{% endstep %}

{% step %}
### What is happening during training?

One full **reinforcement learning training step** in SkyRL looks like:

```bash
(skyrl_entrypoint pid=8455)   Input: [{'content': 'The selling price of a bicycle that had sold for $220 last year was increased by 15%. What is the new price? Let\'s think step by step and output the final answer after "####".', 'role': 'user'}]
(skyrl_entrypoint pid=8455)   Output (Total Reward: 0.0000):
(skyrl_entrypoint pid=8455) To find the new price of the bicycle after a 15% increase from the original price of $220, we can follow these steps:
(skyrl_entrypoint pid=8455)
(skyrl_entrypoint pid=8455) 1. Calculate the amount of the increase by multiplying the original price by 15%. We will use 15% as 0.15 in decimal form, so we calculate:
(skyrl_entrypoint pid=8455)    \[
(skyrl_entrypoint pid=8455)    220 \times 0.15
(skyrl_entrypoint pid=8455)    \]
(skyrl_entrypoint pid=8455)
(skyrl_entrypoint pid=8455) 2. Perform the multiplication:
(skyrl_entrypoint pid=8455)    \[
(skyrl_entrypoint pid=8455)    220 \times 0.15 = 33
(skyrl_entrypoint pid=8455)    \]
(skyrl_entrypoint pid=8455)
(skyrl_entrypoint pid=8455) 3. Add the increase to the original price to find the new price:
(skyrl_entrypoint pid=8455)    \[
(skyrl_entrypoint pid=8455)    220 + 33 = 253
(skyrl_entrypoint pid=8455)    \]
(skyrl_entrypoint pid=8455)
(skyrl_entrypoint pid=8455) Therefore, the new price of the bicycle is **$253**.<|im_end|>
(skyrl_entrypoint pid=8455) 2026-03-26 14:44:59.431 | INFO     | skyrl.train.trainer:train:261 - Started: 'convert_to_training_input'
(skyrl_entrypoint pid=8455) 2026-03-26 14:45:00.969 | INFO     | skyrl.train.trainer:convert_to_training_input:665 - batch_num_seq: 5120, batch_padded_seq_len: 1172
(skyrl_entrypoint pid=8455) 2026-03-26 14:45:00.970 | INFO     | skyrl.train.trainer:convert_to_training_input:683 - Number of sequences before padding: 5120
(skyrl_entrypoint pid=8455) 2026-03-26 14:45:00.970 | INFO     | skyrl.train.trainer:convert_to_training_input:685 - Number of sequences after padding: 5120
(skyrl_entrypoint pid=8455) 2026-03-26 14:45:00.970 | INFO     | skyrl.train.trainer:train:261 - Finished: 'convert_to_training_input', time cost: 1.54s
(skyrl_entrypoint pid=8455) 2026-03-26 14:45:00.970 | INFO     | skyrl.train.trainer:train:265 - Started: 'fwd_logprobs_values_reward'
(skyrl_entrypoint pid=8455) 2026-03-26 14:51:18.325 | INFO     | skyrl.train.trainer:train:265 - Finished: 'fwd_logprobs_values_reward', time cost: 377.36s
(skyrl_entrypoint pid=8455) 2026-03-26 14:51:18.326 | INFO     | skyrl.train.trainer:train:274 - Started: 'compute_advantages_and_returns'
(skyrl_entrypoint pid=8455) 2026-03-26 14:51:18.430 | INFO     | skyrl.train.trainer:compute_advantages_and_returns:873 - avg_final_rewards: 0.13398437201976776, avg_response_length: 273.6802734375
(skyrl_entrypoint pid=8455) 2026-03-26 14:51:18.431 | INFO     | skyrl.train.trainer:train:274 - Finished: 'compute_advantages_and_returns', time cost: 0.11s
(skyrl_entrypoint pid=8455) 2026-03-26 14:51:18.431 | INFO     | skyrl.train.trainer:train:291 - Started: 'train_critic_and_policy'
(skyrl_entrypoint pid=8455) 2026-03-26 14:51:18.431 | INFO     | skyrl.train.trainer:train_critic_and_policy:1121 - Started: 'policy_train'
(skyrl_entrypoint pid=8455) 2026-03-26 15:05:44.917 | INFO     | skyrl.train.trainer:train_critic_and_policy:1121 - Finished: 'policy_train', time cost: 866.49s
(skyrl_entrypoint pid=8455) 2026-03-26 15:05:44.938 | INFO     | skyrl.train.trainer:train:291 - Finished: 'train_critic_and_policy', time cost: 866.51s
(skyrl_entrypoint pid=8455) 2026-03-26 15:05:44.938 | INFO     | skyrl.train.trainer:train:316 - Started: 'sync_weights'
(skyrl_entrypoint pid=8455) 2026-03-26 15:05:51.979 | INFO     | skyrl.train.trainer:train:316 - Finished: 'sync_weights', time cost: 7.04s
(skyrl_entrypoint pid=8455) 2026-03-26 15:05:51.980 | INFO     | skyrl.train.trainer:train:210 - Finished: 'step', time cost: 1331.13s
(skyrl_entrypoint pid=8455) 2026-03-26 15:05:51.980 | INFO     | skyrl.train.trainer:train:320 - {'final_loss': -0.0001395885121230879, 'policy_loss': -0.0001400031556840986, 'policy_entropy': 0.36564826631365577, 'response_length': 1024.0, 'policy_lr': 9.999999974752427e-07, 'loss_metrics/clip_ratio': 0.000960409866502232, 'policy_kl': 0.00041451671746912667, 'grad_norm': 0.3702232241630554}
Training Batches Processed:  14%|█▍        | 1/7 [22:36<2:15:37, 1356.23s/it]
(skyrl_entrypoint pid=8455) 2026-03-26 15:05:52.046 | INFO     | skyrl.train.trainer:train:210 - Started: 'step'
(skyrl_entrypoint pid=8455)
(skyrl_entrypoint pid=8455) 2026-03-26 15:05:52.075 | INFO     | skyrl.train.trainer:train:227 - Started: 'generate'
(skyrl_entrypoint pid=8455) %|          | 0/5120 [00:00<?, ?it/s]
(skyrl_entrypoint pid=8455) %|█         | 512/5120 [00:10<01:37, 47.35it/s]
(skyrl_entrypoint pid=8455) %|██        | 1024/5120 [00:22<01:32, 44.08it/s]
(skyrl_entrypoint pid=8455) %|███       | 1536/5120 [00:28<01:01, 58.46it/s]
(skyrl_entrypoint pid=8455) %|████      | 2048/5120 [00:37<00:53, 57.86it/s]
(skyrl_entrypoint pid=8455) %|█████     | 2581/5120 [00:42<00:36, 68.80it/s]
(skyrl_entrypoint pid=8455) %|██████    | 3093/5120 [00:51<00:31, 64.64it/s]
(skyrl_entrypoint pid=8455) %|███████   | 3615/5120 [00:56<00:20, 72.97it/s]
(skyrl_entrypoint pid=8455) %|████████  | 4127/5120 [01:05<00:14, 67.50it/s]
Generating Trajectories: 100%|██████████| 5120/5120 [01:18<00:00, 65.35it/s]
(skyrl_entrypoint pid=8455) 2026-03-26 15:07:10.550 | INFO     | skyrl.train.trainer:train:227 - Finished: 'generate', time cost: 78.47s
(skyrl_entrypoint pid=8455) 2026-03-26 15:07:10.654 | INFO     | skyrl.train.trainer:train:248 - Started: 'postprocess_generator_output'
(skyrl_entrypoint pid=8455) 2026-03-26 15:07:10.801 | INFO     | skyrl.train.trainer:postprocess_generator_output:776 - reward/avg_pass_at_5: 0.6533203125, reward/avg_raw_reward: 0.2173828125, reward/mean_positive_reward: 0.2173828125
(skyrl_entrypoint pid=8455) 2026-03-26 15:07:10.801 | INFO     | skyrl.train.trainer:train:248 - Finished: 'postprocess_generator_output', time cost: 0.15s
(skyrl_entrypoint pid=8455) 2026-03-26 15:07:10.802 | INFO     | skyrl.train.utils.logging_utils:log_example:81 - Example:
```

At a high level, SkyRL is doing this loop over and over:

1. **Generate answers** with the current model
2. **Score those answers** with the task reward
3. **Turn them into training data**
4. **Compute policy/value statistics**
5. **Update the model**
6. **Sync the updated weights** back to the inference engine
7. **Start the next step**

***

#### 1. The model first generates an answer

The log starts with a concrete example prompt:

```
Input: [{'content': 'The selling price of a bicycle that had sold for $220 last year was increased by 15%. What is the new price? ...', 'role': 'user'}]
```

Then SkyRL shows one sampled output from the model:

```
Therefore, the new price of the bicycle is **$253**.<|im_end|>
```

This means the current policy model has already done rollout generation for that prompt. In other words, the model is not being trained on a fixed answer here — it is **producing its own answer first**, and that answer becomes part of the RL batch.

So in plain English:

* the model saw a math word problem
* it generated a chain-of-thought style response
* that generated response is now part of the training material for this step

***

#### 2. SkyRL converts the generated outputs into training tensors

Next you see:

```
Started: 'convert_to_training_input'
batch_num_seq: 5120, batch_padded_seq_len: 1172
Number of sequences before padding: 5120
Number of sequences after padding: 5120
Finished: 'convert_to_training_input', time cost: 1.54s
```

This stage takes all of the newly generated samples and prepares them for training.

What that means in practice:

* SkyRL has collected **5120 generated sequences**
* each sequence is a prompt plus a model-generated response
* they are padded to a common length, here **1172 tokens**
* then they are packed into tensors that the trainer can feed into the model

So this is the “data formatting” phase of the RL step.

A good mental model is:

> generation produces raw text, and `convert_to_training_input` turns that raw text into something the optimizer can use

***

#### 3. SkyRL computes logprobs, values, and rewards

Then training enters one of the expensive stages:

```
Started: 'fwd_logprobs_values_reward'
Finished: 'fwd_logprobs_values_reward', time cost: 377.36s
```

This is where the trainer runs forward passes to compute the quantities needed for RL optimization.

That stage typically includes:

* **logprobs**: how likely the policy thought each generated token was
* **values**: the value function / critic estimate for the trajectory
* **reward-related information**: the signals used to determine whether the output was good or bad

This phase took **377.36 seconds**, which tells you it is one of the heavy parts of the step.

That makes sense because the batch is large:

* **5120 sequences**
* padded to **1172 tokens**

So the trainer is doing a lot of token-level computation here.

***

#### 4. SkyRL computes advantages and returns

Next:

```
Started: 'compute_advantages_and_returns'
avg_final_rewards: 0.13398437201976776, avg_response_length: 273.6802734375
Finished: 'compute_advantages_and_returns', time cost: 0.11s
```

This is the stage where RL training targets are prepared.

The important number here is:

```
avg_final_rewards: 0.13398437201976776
```

That means the average reward across this batch was about **0.134**.

The other useful number is:

```
avg_response_length: 273.68
```

So even though sequences were padded to a much larger common length for batching, the average actual generated response was around **274 tokens**.

Conceptually, this step answers:

* which sampled outputs were better?
* by how much?
* what learning signal should the optimizer use?

This stage is fast because the heavy model forward work already happened in the previous phase.

***

#### 5. SkyRL updates the policy

Then the trainer starts the actual model update:

```
Started: 'train_critic_and_policy'
Started: 'policy_train'
Finished: 'policy_train', time cost: 866.49s
Finished: 'train_critic_and_policy', time cost: 866.51s
```

This is the point where the policy is actually being trained.

The important thing to understand is:

* the model has already generated outputs
* those outputs have already been scored
* advantages have already been computed
* now the optimizer uses that information to **change the model weights**

This took **866.49 seconds**, which is even longer than the forward-statistics phase. So in this run, the biggest chunk of time in the step is the actual policy update.

In human terms:

> this is where the model learns from the answers it just produced

***

#### 6. The updated weights are synced back to inference

After the policy update, you see:

```
Started: 'sync_weights'
Finished: 'sync_weights', time cost: 7.04s
```

SkyRL uses a training stack and an inference stack together. After the policy changes, the inference engine needs the newest weights so the next rollout uses the updated model instead of the old one.

That is what this sync step is doing.

It is short compared with the training phases, but it is important because it closes the loop between:

* training the model
* using the updated model to generate the next batch

***

#### 7. One full RL step is now complete

Then SkyRL reports:

```
Finished: 'step', time cost: 1331.13s
```

So one complete RL step took:

* about **1331 seconds**
* roughly **22.2 minutes**

That full step included:

* converting generated outputs into training input
* computing logprobs / values / rewards
* computing advantages and returns
* training the policy
* syncing the updated weights

Immediately after that, SkyRL prints training metrics:

```
{'final_loss': -0.0001395885121230879,
 'policy_loss': -0.0001400031556840986,
 'policy_entropy': 0.36564826631365577,
 'response_length': 1024.0,
 'policy_lr': 9.999999974752427e-07,
 'loss_metrics/clip_ratio': 0.000960409866502232,
 'policy_kl': 0.00041451671746912667,
 'grad_norm': 0.3702232241630554}
```

These are the summary numbers for that training update.

And then the progress bar confirms the trainer has completed **1 out of 7** batches:

```
Training Batches Processed:  14%|█▍        | 1/7
```

So at this point:

* the first RL training step is done
* the model has been updated once
* training is moving on to step 2

**On the next step, the run showed higher reward metrics:**

```
reward/avg_pass_at_5: 0.6533203125
reward/avg_raw_reward: 0.2173828125
```

Compared to the earlier step:

```
reward/avg_pass_at_5: 0.4912109375
reward/avg_raw_reward: 0.133984375
```

That is a useful sign that training is not just running — it is moving in a promising direction.
{% endstep %}
{% endstepper %}

***

## Useful output locations

During the run, SkyRL wrote outputs to a few standard locations.

### Checkpoints

```bash
/home/ray/ckpts/
```

### Exported evaluation results

```bash
/home/ray/exports/dumped_evals/
```

For example:

```bash
/home/ray/exports/dumped_evals/global_step_0_evals/openai_gsm8k.jsonl
/home/ray/exports/dumped_evals/global_step_0_evals/aggregated_results.jsonl
```

### Infrastructure logs

```bash
/tmp/skyrl-logs/
```

These are useful if you want to inspect trainer or Ray-side behavior later.
