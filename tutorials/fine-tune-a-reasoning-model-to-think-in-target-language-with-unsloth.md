# Fine-tune a Reasoning Model to Think in Target Language with Unsloth

> 🦥 This tutorial is created based on [Unsloth official notebooks](https://unsloth.ai/docs/get-started/unsloth-notebooks).

**Model:** DeepSeek-R1-0528-Qwen3-8B (4-bit QLoRA + vLLM fast inference)\
**Algorithm:** GRPO (Group Relative Policy Optimization)\
**Dataset:** [DAPO-Math-17k](https://huggingface.co/datasets/open-r1/DAPO-Math-17k-Processed)

***

Most GRPO tutorials only reward correctness. This notebook adds a language-consistency reward — inspired directly by the DeepSeek-R1 paper — that forces the model's internal `<think>` reasoning chain to use Bahasa Indonesia. The technique generalises to any target language.

```
Reward signal breakdown (max total = 16.5 per step):
  match_format_exactly       +3.0  — <think>...</think> tags present and correct
  match_format_approximately +1.0  — partial credit for tag structure
  check_answer               +5.0  — exact answer match (with ratio-based partial credit)
  check_numbers              +3.5  — float comparison with comma normalisation
  language_consistency       +5.0  — reasoning trace detected as Bahasa Indonesia
```

***

## Section 1 — YottaLabs Platform

Before running any code in this notebook, you need to spin up a virtual machine on YottaLabs.

### Recommended GPU Configuration

| GPU           | VRAM  | Speed vs A100 | Recommended? | Note                                 |
| ------------- | ----- | ------------- | ------------ | ------------------------------------ |
| **H100 80GB** | 80 GB | 2×            | ⭐ **Best**   | Plenty of headroom, fastest training |

> **TL;DR:** Use **1× H100 80GB**. With 4-bit QLoRA the model uses \~25 GB, giving you lots of room.

### Launch the VM

To get started with our VM, see our [official doc](https://docs.yottalabs.ai/products/virtual-machines/launching-a-virtual-machine).

***

## Section 2 — Environment Setup

{% hint style="info" %}
Run all cells from here inside your YottaLabs Pod via Jupyter.
{% endhint %}

### Key Differences from Standard Unsloth Setup

This notebook uses **vLLM** for fast GRPO rollout generation. vLLM dramatically speeds up the `num_generations` sampling step — instead of running the model sequentially for each rollout, vLLM batches them with continuous batching and PagedAttention.

Additionally, we install `langid` — a fast language identification library used in our language-consistency reward function.

| Package        | Version | Why pinned                                              |
| -------------- | ------- | ------------------------------------------------------- |
| `transformers` | 4.56.2  | Compatible with DeepSeek-R1 tokenizer + Unsloth patches |
| `trl`          | 0.22.2  | GRPO trainer version tested with vLLM integration       |
| `vllm`         | auto    | Fast inference engine for GRPO rollouts                 |
| `langid`       | latest  | Language detection for the consistency reward           |

```python
%%capture
import os
os.environ["UNSLOTH_VLLM_STANDBY"] = "1" # [NEW] Extra 30% context lengths!
!pip install unsloth vllm
```

```python
%pip install langid -qq
```

***

## Section 3 — Load DeepSeek-R1-0528-Qwen3-8B

### Why `fast_inference = True`?

Unlike the Vision GRPO notebook, this text-only model enables **vLLM fast inference** during training. This is possible because:

1. vLLM is installed and compatible with this model architecture
2. Text-only GRPO rollouts can be batched efficiently with vLLM's continuous batching
3. On an H100, vLLM generates \~4 rollouts in roughly the same time standard generation produces 1

### LoRA Configuration

We use `lora_alpha = rank * 2` here (alpha = 64 for rank 32). This is a common trick that effectively **doubles the learning rate for LoRA layers** without changing the base LR, leading to faster convergence in RL settings.

| Parameter                | Value       | Why                                                                      |
| ------------------------ | ----------- | ------------------------------------------------------------------------ |
| `lora_rank`              | 32          | Higher than vision notebook — text reasoning benefits from more capacity |
| `lora_alpha`             | 64 (rank×2) | 2× alpha speeds up LoRA convergence                                      |
| `max_seq_length`         | 1024        | Short for speed; increase to 4096+ for longer reasoning                  |
| `fast_inference`         | True        | Enable vLLM — critical for GRPO rollout speed                            |
| `gpu_memory_utilization` | 0.9         | High — vLLM needs KV cache VRAM                                          |

```python
from unsloth import FastLanguageModel
import torch
max_seq_length = 1024 # Can increase for longer reasoning traces
lora_rank = 32 # Larger rank = smarter, but slower

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name = "unsloth/DeepSeek-R1-0528-Qwen3-8B",
    max_seq_length = max_seq_length,
    load_in_4bit = True, # False for LoRA 16bit
    fast_inference = True, # Enable vllm fast inference
    max_lora_rank = lora_rank,
    gpu_memory_utilization = 0.9, # Reduce if out of memory
)

model = FastLanguageModel.get_peft_model(
    model,
    r = lora_rank, # Choose any number > 0 ! Suggested 8, 16, 32, 64, 128
    target_modules = [
        "q_proj", "k_proj", "v_proj", "o_proj",
        "gate_proj", "up_proj", "down_proj",
    ],
    lora_alpha = lora_rank*2, # *2 speeds up training
    use_gradient_checkpointing = "unsloth", # Reduces memory usage
    random_state = 3407,
)
```

***

## Section 4 — Understanding the DeepSeek-R1 Chat Template

DeepSeek-R1 models use special tokens to delimit the **internal reasoning chain** from the **final answer**. The GRPOTrainer automatically prepends the `<think>` opening token to every completion, so the model always starts in reasoning mode.

### Token Discovery

We programmatically find the special tokens rather than hardcoding them, making this code robust across model variants:

```python
reasoning_start = None
reasoning_end = None
user_token = None
assistant_token = None

for token in tokenizer.get_added_vocab().keys():
    if "think" in token and "/" in token:
        reasoning_end = token
    elif "think" in token:
        reasoning_start = token
    elif "user" in token:
        user_token = token
    elif "assistant" in token:
        assistant_token = token

system_prompt = \
f"""You are given a problem.
Think about the problem and provide your working out.
You must think in Bahasa Indonesia."""
system_prompt
```

### System Prompt for Language Targeting

We instruct the model to reason in **Bahasa Indonesia** via the system prompt. The language-consistency reward function (Section 6) then reinforces this behaviour during training.

{% hint style="info" %}
Design principle: the system prompt _asks_ for the target language; the reward function _enforces_ it. Neither alone is sufficient — both are needed for reliable steering.
{% endhint %}

```python
print(tokenizer.apply_chat_template([
    {"role" : "user", "content" : "What is 1+1?"},
    {"role" : "assistant", "content" : f"<think>I think it's 2.2</think>2"},
    {"role" : "user", "content" : "What is 1+1?"},
    {"role" : "assistant", "content" : f"<think>I think it's 2.2</think>2"},
], tokenize = False, add_generation_prompt = True))
```

***

## Section 5 — Prepare the DAPO-Math Dataset

We use [DAPO-Math-17k](https://huggingface.co/datasets/open-r1/DAPO-Math-17k-Processed) — a curated math reasoning dataset from the Open-R1 project, containing 17k problems with ground-truth solutions compatible with GRPO training.

### Why Filter by Token Length?

GRPO requires every prompt in a batch to fit within `max_prompt_length`. Rather than truncating (which corrupts problems), we **remove the top 10% longest prompts**. This ensures:

* No silent truncation of math problem statements
* Predictable memory usage per step
* Faster per-step time (no padding to extreme lengths)

### Data Pipeline

```python
from datasets import load_dataset
dataset = load_dataset("open-r1/DAPO-Math-17k-Processed", "en", split = "train")
dataset
```

```python
dataset[0]["prompt"]
```

```python
dataset[0]["solution"]
```

```python
def extract_hash_answer(text):
    # if "####" not in text: return None
    # return text.split("####")[1].strip()
    return text
extract_hash_answer(dataset[0]["solution"])
```

```python
dataset = dataset.map(lambda x: {
    "prompt" : [
        {"role": "system", "content": system_prompt},
        {"role": "user",   "content": x["prompt"]},
    ],
    "answer": extract_hash_answer(x["solution"]),
})
dataset[0]
```

```python
tokenized = dataset.map(
    lambda x: {"tokens" : tokenizer.apply_chat_template(x["prompt"], add_generation_prompt = True, tokenize = True)},
    batched = True,
)
print(tokenizer.decode(tokenized[0]["tokens"]))
tokenized = tokenized.map(lambda x: {"L" : len(x["tokens"])})

import numpy as np
maximum_length = int(np.quantile(tokenized["L"], 0.9))
print("Max Length = ", maximum_length)

# Filter only samples smaller than 90% max length
dataset = dataset.select(np.where(np.array(tokenized["L"]) <= maximum_length)[0])
del tokenized
```

***

## Section 6 — Five Reward Functions

This notebook uses **five complementary reward functions** that together guide the model towards structured, correct, and Indonesian-language reasoning. Each targets a different aspect of quality.

### Reward Architecture Overview

```
Completion
    │
    ├─► match_format_exactly       → Does </think> appear? (+3.0)
    ├─► match_format_approximately → Are <think>/</think> counts right? (+1.0 max)
    ├─► check_answer               → Does extracted answer match ground truth? (+5.0 max)
    ├─► check_numbers              → Does extracted float match ground truth? (+3.5 max)
    └─► language_consistency       → Is the reasoning in Bahasa Indonesia? (+5.0)
                                                            ───────────────
                                                  Max total: +17.5 per completion
```

### Why Multiple Rewards?

* `match_format_exactly` gives a strong binary signal for correct structure
* `match_format_approximately` provides gradient even for near-miss formatting
* `check_answer` rewards exact string matches and ratio-based proximity
* `check_numbers` catches cases where the answer is embedded in a sentence
* `language_consistency` is the novel reward that steers the reasoning language

Using multiple rewards is better than one composite reward because each signal is cleaner and easier to tune independently.

```python
import re

# Add optional EOS token matching
solution_end_regex = rf"{reasoning_end}(.*)"

match_format = re.compile(solution_end_regex, re.DOTALL)
match_format
```

```python
match_format.findall(
    "Let me think!</think>"\
    f"Hence, the solution is 2.",
)
```

```python
match_format.findall(
    "<think>Let me think!</think>"\
    f"\n\nHence, the solution is 2",
)
```

```python
def match_format_exactly(completions, **kwargs):
    scores = []
    for completion in completions:
        score = 0
        response = completion[0]["content"]
        # Match if format is seen exactly!
        if match_format.search(response) is not None: score += 3.0
        scores.append(score)
    return scores
```

```python
def match_format_approximately(completions, **kwargs):
    scores = []
    for completion in completions:
        score = 0
        response = completion[0]["content"]
        # Count how many keywords are seen - we penalize if too many!
        # If we see 1, then plus some points!

        # No need to reward <think> since we always prepend it!
        score += 0.5 if response.count(reasoning_start) == 1 else -1.0
        score += 0.5 if response.count(reasoning_end)   == 1 else -1.0
        scores.append(score)
    return scores
```

```python
def check_answer(prompts, completions, answer, **kwargs):
    question = prompts[0][-1]["content"]
    responses = [completion[0]["content"] for completion in completions]

    extracted_responses = [
        guess.group(1)
        if (guess := match_format.search(r)) is not None else None \
        for r in responses
    ]

    scores = []
    for guess, true_answer in zip(extracted_responses, answer):
        score = 0
        if guess is None:
            scores.append(-2.0)
            continue
        # Correct answer gets 5 points!
        if guess == true_answer:
            score += 5.0
        # Match if spaces are seen, but less reward
        elif guess.strip() == true_answer.strip():
            score += 3.5
        else:
            # We also reward it if the answer is close via ratios!
            # Ie if the answer is within some range, reward it!
            try:
                ratio = float(guess) / float(true_answer)
                if   ratio >= 0.9 and ratio <= 1.1: score += 2.0
                elif ratio >= 0.8 and ratio <= 1.2: score += 1.5
                else: score -= 2.5 # Penalize wrong answers
            except:
                score -= 4.5 # Penalize
        scores.append(score)
    return scores
```

```python
match_numbers = re.compile(
    r".*?[\s]{0,}([-]?[\d\.\,]{1,})",
    flags = re.MULTILINE | re.DOTALL
)
print(match_numbers.findall("  0.34  "))
print(match_numbers.findall("  123,456  "))
print(match_numbers.findall("  -0.234  "))
print(match_numbers.findall("17"))
```

```python
import langid

def get_lang(text: str) -> str:
    if not text:
        return "und"
    lang, _ = langid.classify(text)
    return lang


print(get_lang("Hello, How are you")) # This should return en
print(get_lang("Aku berpikir kalau aku adalah kamu")) # This should return id
print(get_lang("我在这里")) # This should return zh
```

```python
import re

def format_and_language_reward_func(completions, **kwargs):
    scores = []

    for completion_item in completions:
        if not completion_item or not isinstance(completion_item[0], dict) or "content" not in completion_item[0]:
            scores.append(-5.0)
            print(f"Warning: Malformed completion item, assigning default low score: {completion_item}")
            continue

        content = completion_item[0]["content"]

        lang = get_lang(content)

        if lang == 'id':
            score = 5.0
        elif lang == 'en':
            score = -3.0
        elif lang == 'zh':
            score = -3.0
        else:
            score = -5.0

        scores.append(score)

    return scores
```

```python
prompts = [
    [{"role": "assistant", "content": "What is the result of (1 + 2) * 4?"}],
    [{"role": "assistant", "content": "What is the result of (3 + 1) * 2?"}],
]
completions = [
    [{"role": "assistant", "content": "<think>The sum of 1 and 2 is 3, which we multiply by 4 to get 12.</think><answer>(1 + 2) * 4 = 12</answer>"}],
    [{"role": "assistant", "content": "The sum of 3 and 1 is 4, which we multiply by 2 to get 8. So (3 + 1) * 2 = 8."}],
]
format_and_language_reward_func(prompts = prompts, completions = completions)
```

```python
global PRINTED_TIMES
PRINTED_TIMES = 0
global PRINT_EVERY_STEPS
PRINT_EVERY_STEPS = 5

def check_numbers(prompts, completions, answer, **kwargs):
    question = prompts[0][-1]["content"]
    responses = [completion[0]["content"] for completion in completions]

    extracted_responses = [
        guess.group(1)
        if (guess := match_numbers.search(r)) is not None else None \
        for r in responses
    ]

    scores = []
    # Print only every few steps
    global PRINTED_TIMES
    global PRINT_EVERY_STEPS
    if PRINTED_TIMES % PRINT_EVERY_STEPS == 0:
        print(
            '*'*20 + f"Question:\n{question}", f"\nAnswer:\n{answer[0]}", f"\nResponse:\n{responses[0]}", f"\nExtracted:\n{extracted_responses[0]}"
        )
    PRINTED_TIMES += 1

    for guess, true_answer in zip(extracted_responses, answer):
        if guess is None:
            scores.append(-2.5)
            continue
        # Convert to numbers
        try:
            true_answer = float(true_answer.strip())
            # Remove commas like in 123,456
            guess       = float(guess.strip().replace(",", ""))
            scores.append(3.5 if guess == true_answer else -1.5)
        except:
            scores.append(0)
            continue
    return scores
```

***

## Section 7 — Configure & Launch GRPO Training

### vLLM Integration

When `fast_inference=True`, GRPOTrainer routes all rollout generation through vLLM's `SamplingParams`. This gives a significant speedup on H100:

| Method                        | 4 rollouts time (approx) |
| ----------------------------- | ------------------------ |
| Standard HuggingFace generate | \~8 seconds              |
| vLLM with PagedAttention      | \~2 seconds              |

We configure `SamplingParams` separately from `GRPOConfig` — vLLM uses its own sampling API.

### Sequence Length Calculation

We compute `max_completion_length` dynamically from the measured max prompt length, ensuring the total sequence always fits within `max_seq_length`. This prevents silent truncation of reasoning chains, which would produce noisy gradients.

### Training Duration

We run for **100 steps** here (`max_steps=100`) — enough to see rewards start climbing, and to verify the training loop works. For production quality, train for 1–3 full epochs (`num_train_epochs=1`, remove `max_steps`).

> ℹ️ Watch the `reward` and `rewards/language` columns in the training table. You expect `reward` to climb from \~0 to \~2+ after 100 steps, and the language reward to become increasingly positive as the model adopts Indonesian.

```python
max_prompt_length = maximum_length + 1 # + 1 just in case!
max_completion_length = max_seq_length - max_prompt_length

from vllm import SamplingParams
vllm_sampling_params = SamplingParams(
    min_p = 0.1,
    top_p = 1.0,
    top_k = -1,
    seed = 3407,
    stop = [tokenizer.eos_token],
    include_stop_str_in_output = True,
)

from trl import GRPOConfig, GRPOTrainer
training_args = GRPOConfig(
    vllm_sampling_params = vllm_sampling_params,
    temperature = 1.0,
    learning_rate = 5e-6,
    weight_decay = 0.001,
    warmup_ratio = 0.1,
    lr_scheduler_type = "linear",
    optim = "adamw_8bit",
    logging_steps = 1,
    per_device_train_batch_size = 1,
    gradient_accumulation_steps = 1, # Increase to 4 for smoother training
    num_generations = 4, # Decrease if out of memory
    max_prompt_length = max_prompt_length,
    max_completion_length = max_completion_length,
    # num_train_epochs = 1, # Set to 1 for a full training run
    max_steps = 100,
    save_steps = 100,
    report_to = "none", # Can use Weights & Biases
    output_dir = "outputs",

    # For optional training + evaluation
    # fp16_full_eval = True,
    # per_device_eval_batch_size = 4,
    # eval_accumulation_steps = 1,
    # eval_strategy = "steps",
    # eval_steps = 1,
)
```

```python
# For optional training + evaluation
# new_dataset = dataset.train_test_split(test_size = 0.01)

trainer = GRPOTrainer(
    model = model,
    processing_class = tokenizer,
    reward_funcs = [
        match_format_exactly,
        match_format_approximately,
        check_answer,
        check_numbers,
        format_and_language_reward_func,
    ],
    args = training_args,
    train_dataset = dataset,

    # For optional training + evaluation
    # train_dataset = new_dataset["train"],
    # eval_dataset = new_dataset["test"],
)
trainer.train()
```

***

## Section 8 — Inference & Language Compliance Evaluation

We run three comparison experiments:

1. **Without LoRA** — baseline model behaviour (no system prompt)
2. **With LoRA, no system prompt** — does LoRA affect general reasoning?
3. **With LoRA + system prompt** — the intended deployment mode (Indonesian reasoning)

Then we run a **batch evaluation on 20 samples** to measure the language compliance rate quantitatively.

```python
text = "What is the sqrt of 101?"

from vllm import SamplingParams
sampling_params = SamplingParams(
    temperature = 1.0,
    top_k = 50,
    max_tokens = 1024,
)
output = model.fast_generate(
    [text],
    sampling_params = sampling_params,
    lora_request = None,
)[0].outputs[0].text

output
```

```python
model.save_lora("grpo_lora")
```

{% hint style="info" %}
`model.save_lora()` is used here instead of `model.save_pretrained()` because we enabled `fast_inference=True` (vLLM mode). The vLLM-backed model uses the Unsloth `save_lora` API.
{% endhint %}

```python
from safetensors import safe_open

tensors = {}
with safe_open("grpo_lora/adapter_model.safetensors", framework = "pt") as f:
    # Verify both A and B are non zero
    for key in f.keys():
        tensor = f.get_tensor(key)
        n_zeros = (tensor == 0).sum() / tensor.numel()
        assert(n_zeros.item() != tensor.numel())
```

```python
messages = [
    {"role": "user",   "content": "Solve (x + 2)^2 = 0"},
]

text = tokenizer.apply_chat_template(
    messages,
    add_generation_prompt = True, # Must add for generation
    tokenize = False,
)
from vllm import SamplingParams
sampling_params = SamplingParams(
    temperature = 1.0,
    top_k = 50,
    max_tokens = 2048,
)
output = model.fast_generate(
    text,
    sampling_params = sampling_params,
    lora_request = model.load_lora("grpo_lora"),
)[0].outputs[0].text

output
```

```python
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user",   "content": "Solve (x + 2)^2 = 0"},
]

text = tokenizer.apply_chat_template(
    messages,
    add_generation_prompt = True, # Must add for generation
    tokenize = False,
)
from vllm import SamplingParams
sampling_params = SamplingParams(
    temperature = 1.0,
    top_k = 50,
    max_tokens = 2048,
)
output = model.fast_generate(
    text,
    sampling_params = sampling_params,
    lora_request = model.load_lora("grpo_lora"),
)[0].outputs[0].text

output
```

```python
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user",   "content": "Solve (x + 2)^2 = 0"},
]

text = tokenizer.apply_chat_template(
    messages,
    add_generation_prompt = True, # Must add for generation
    tokenize = False,
)
from vllm import SamplingParams
sampling_params = SamplingParams(
    temperature = 1.0,
    top_k = 50,
    max_tokens = 2048,
)
output = model.fast_generate(
    text,
    sampling_params = sampling_params,
    lora_request = None,
)[0].outputs[0].text

output
```

### Batch Language Compliance Evaluation

A single example is anecdotal. We run over **20 randomly sampled problems** and measure the percentage that produce Indonesian reasoning chains — with and without the LoRA.

```python
sample_dataset = dataset.shuffle(seed = 3407).select(range(20))
sample_dataset
```

```python
with_lora_id_count = 0
without_lora_id_count = 0

print("Comparing language usage with and without LoRA on 20 samples:")
print("=" * 60)

for i, sample in enumerate(sample_dataset):
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": sample["prompt"][1]["content"]},
    ]

    text = tokenizer.apply_chat_template(
        messages,
        add_generation_prompt = True,
        tokenize = False,
    )

    output_with_lora = model.fast_generate(
        text,
        sampling_params = sampling_params,
        lora_request = model.load_lora("grpo_lora"),
    )[0].outputs[0].text

    output_without_lora = model.fast_generate(
        text,
        sampling_params = sampling_params,
        lora_request = None,
    )[0].outputs[0].text

    lang_with_lora = get_lang(output_with_lora)
    lang_without_lora = get_lang(output_without_lora)

    if lang_with_lora == 'id':
        with_lora_id_count += 1
    if lang_without_lora == 'id':
        without_lora_id_count += 1

    # Print progress every 5 samples
    if (i + 1) % 5 == 0:
        print(f"Processed {i + 1}/20 samples...")

print("\n" + "=" * 60)
print("RESULTS:")
print(f"With LoRA - Indonesian responses: {with_lora_id_count}/20 ({with_lora_id_count/20*100:.1f}%)")
print(f"Without LoRA - Indonesian responses: {without_lora_id_count}/20 ({without_lora_id_count/20*100:.1f}%)")
print(f"Improvement: +{with_lora_id_count - without_lora_id_count} Indonesian responses with LoRA")
```

***

## Section 9 — Save & Export

### Export Options

| Method            | Size     | Use Case                                                    |
| ----------------- | -------- | ----------------------------------------------------------- |
| **LoRA adapters** | \~200 MB | Share on HuggingFace, continue training, hot-swap with vLLM |
| **Merged 16-bit** | \~16 GB  | Deploy with vLLM as a standalone model                      |
| **Merged 4-bit**  | \~5 GB   | Memory-constrained vLLM deployment                          |
| **GGUF q4\_k\_m** | \~5 GB   | llama.cpp / Ollama local inference                          |
| **GGUF q8\_0**    | \~9 GB   | High-quality llama.cpp inference                            |

> **Note:** `model.save_lora()` is used here instead of `model.save_pretrained()` because we enabled `fast_inference=True` (vLLM mode). The vLLM-backed model uses the Unsloth `save_lora` API.

```python
# Merge to 16bit
if False: model.save_pretrained_merged("deepseek_r1_finetune_16bit", tokenizer, save_method = "merged_16bit",)
if False: model.push_to_hub_merged("HF_USERNAME/deepseek_r1_finetune_16bit", tokenizer, save_method = "merged_16bit", token = "YOUR_HF_TOKEN")

# Merge to 4bit
if False: model.save_pretrained_merged("deepseek_r1_finetune_4bit", tokenizer, save_method = "merged_4bit",)
if False: model.push_to_hub_merged("HF_USERNAME/deepseek_r1_finetune_4bit", tokenizer, save_method = "merged_4bit", token = "YOUR_HF_TOKEN")

# Just LoRA adapters
if False:
    model.save_pretrained("deepseek_r1_lora")
    tokenizer.save_pretrained("deepseek_r1_lora")
if False:
    model.push_to_hub("HF_USERNAME/deepseek_r1_lora", token = "YOUR_HF_TOKEN")
    tokenizer.push_to_hub("HF_USERNAME/deepseek_r1_lora", token = "YOUR_HF_TOKEN")
```

### GGUF / llama.cpp Conversion

To save to `GGUF` / `llama.cpp`, we support it natively now! We clone `llama.cpp` and we default save it to `q8_0`. We allow all methods like `q4_k_m`. Use `save_pretrained_gguf` for local saving and `push_to_hub_gguf` for uploading to HF.

Some supported quant methods (full list on our [docs page](https://unsloth.ai/docs/basics/inference-and-deployment/saving-to-gguf)):

* `q8_0` - Fast conversion. High resource use, but generally acceptable.
* `q4_k_m` - Recommended. Uses Q6\_K for half of the attention.wv and feed\_forward.w2 tensors, else Q4\_K.
* `q5_k_m` - Recommended. Uses Q6\_K for half of the attention.wv and feed\_forward.w2 tensors, else Q5\_K.

```python
# Save to 8bit Q8_0
if False: model.save_pretrained_gguf("deepseek_r1_finetune", tokenizer,)
# Remember to go to https://huggingface.co/settings/tokens for a token!
# And change hf to your username!
if False: model.push_to_hub_gguf("HF_USERNAME/deepseek_r1_finetune", tokenizer, token = "YOUR_HF_TOKEN")

# Save to 16bit GGUF
if False: model.save_pretrained_gguf("deepseek_r1_finetune", tokenizer, quantization_method = "f16")
if False: model.push_to_hub_gguf("HF_USERNAME/deepseek_r1_finetune", tokenizer, quantization_method = "f16", token = "YOUR_HF_TOKEN")

# Save to q4_k_m GGUF
if False: model.save_pretrained_gguf("deepseek_r1_finetune", tokenizer, quantization_method = "q4_k_m")
if False: model.push_to_hub_gguf("HF_USERNAME/deepseek_r1_finetune", tokenizer, quantization_method = "q4_k_m", token = "YOUR_HF_TOKEN")

# Save to multiple GGUF options - much faster if you want multiple!
if False:
    model.push_to_hub_gguf(
        "HF_USERNAME/deepseek_r1_finetune", # Change hf to your username!
        tokenizer,
        quantization_method = ["q4_k_m", "q8_0", "q5_k_m",],
        token = "YOUR_HF_TOKEN",
    )
```

Now, use the `deepseek_r1_finetune.Q8_0.gguf` file or `deepseek_r1_finetune.Q4_K_M.gguf` file in llama.cpp.
