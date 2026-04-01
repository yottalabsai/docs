---
icon: whale
---

# Fine-tune a Reasoning Model to Think in Target Language with Unsloth

> :sloth:This tutorial is created based on [Unsloth official notebooks](https://unsloth.ai/docs/get-started/unsloth-notebooks).&#x20;

**Model:** DeepSeek-R1-0528-Qwen3-8B (4-bit QLoRA + vLLM fast inference)\
**Algorithm:** GRPO (Group Relative Policy Optimization)\
**Dataset:** [DAPO-Math-17k](https://huggingface.co/datasets/open-r1/DAPO-Math-17k-Processed)<br>

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

## &#x20;Section 1 — YottaLabs Platform

Before running any code in this notebook, you need to spin up a GPU Pod on YottaLabs.&#x20;

{% stepper %}
{% step %}
#### Recommended GPU Configuration

| GPU           | VRAM  | Speed vs A100 | Recommended? | Note                                 |
| ------------- | ----- | ------------- | ------------ | ------------------------------------ |
| **H100 80GB** | 80 GB | 2×            | ⭐ **Best**   | Plenty of headroom, fastest training |

{% hint style="info" %}
**TL;DR:** Use **1× H100 80GB**. With 4-bit QLoRA the model uses \~25 GB, giving you lots of room.
{% endhint %}
{% endstep %}

{% step %}
#### Launch a Pod

To get started with our pods, see our [official doc](https://docs.yottalabs.ai/products/gpu-pods)
{% endstep %}
{% endstepper %}

## Section 2 — Environment Setup

{% hint style="info" %}
Run all cells from here inside your YottaLabs Pod via Jupyter&#x20;
{% endhint %}

### Key Differences from Standard Unsloth Setup

This notebook uses **vLLM** for fast GRPO rollout generation. vLLM dramatically speeds up the `num_generations` sampling step — instead of running the model sequentially for each rollout, vLLM batches them with continuous batching and PagedAttention.

Additionally, we install **`langid`** — a fast language identification library used in our language-consistency reward function.

| Package        | Version | Why pinned                                              |
| -------------- | ------- | ------------------------------------------------------- |
| `transformers` | 4.56.2  | Compatible with DeepSeek-R1 tokenizer + Unsloth patches |
| `trl`          | 0.22.2  | GRPO trainer version tested with vLLM integration       |
| `vllm`         | auto    | Fast inference engine for GRPO rollouts                 |
| `langid`       | latest  | Language detection for the consistency reward           |

```python
import sys

def pip(*args):
    """Install packages using the current Python interpreter — works in any environment."""
    import subprocess
    return subprocess.run([sys.executable, "-m", "pip"] + list(args), check=False).returncode

print("Step 1/3: Installing Unsloth + vLLM (this may take 3–5 minutes) ...")
pip("install", "-q", "--upgrade", "uv")

# Use uv for fast dependency resolution where possible
import subprocess
subprocess.run(
    f"{sys.executable} -m uv pip install -qqq --upgrade "
    "vllm bitsandbytes xformers unsloth torchvision pillow",
    shell=True
)

print("Step 2/3: Pinning transformers and trl for DeepSeek-R1 compatibility ...")
pip("install", "-q", "transformers==4.56.2")
pip("install", "-q", "--no-deps", "trl==0.22.2")

print("Step 3/3: Installing langid for language detection reward ...")
pip("install", "-q", "langid")

print("\n✅ All dependencies installed successfully!")
```

```python
# Verify the environment
import torch, langid

assert torch.cuda.is_available(), "❌ No GPU — check your Pod configuration!"

gpu_name = torch.cuda.get_device_name(0)
vram_gb  = torch.cuda.get_device_properties(0).total_memory / 1e9

print(f"✅ GPU:        {gpu_name}")
print(f"   VRAM:       {vram_gb:.1f} GB")
print(f"   PyTorch:    {torch.__version__}")
print(f"   CUDA:       {torch.version.cuda}")

# Quick langid sanity check
lang, _ = langid.classify("Hello world")
assert lang == "en", f"langid sanity check failed: got {lang}"
print(f"\n✅ langid working correctly (detected 'Hello world' as: {lang})")

if vram_gb < 40:
    print("\n⚠️  WARNING: Less than 40 GB VRAM. Set gpu_memory_utilization=0.7 and num_generations=2.")
```

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
import os
os.environ["UNSLOTH_VLLM_STANDBY"] = "1"  # Enables ~30% longer context windows with vLLM Standby

from unsloth import FastLanguageModel
import torch

# ── Configuration ─────────────────────────────────────────────────────────────
MAX_SEQ_LENGTH = 1024   # Increase to 4096+ for longer/more complex reasoning chains
LORA_RANK      = 32     # Higher rank = more expressive adapters

print("Loading DeepSeek-R1-0528-Qwen3-8B (~5 GB download, ~2 min) ...")

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name             = "unsloth/DeepSeek-R1-0528-Qwen3-8B",
    max_seq_length         = MAX_SEQ_LENGTH,
    load_in_4bit           = True,          # QLoRA: reduces model to ~5 GB VRAM
    fast_inference         = True,          # ← Enable vLLM for fast GRPO rollouts
    max_lora_rank          = LORA_RANK,
    gpu_memory_utilization = 0.9,           # High — vLLM KV cache needs space
)

model = FastLanguageModel.get_peft_model(
    model,
    r              = LORA_RANK,
    target_modules = [
        "q_proj", "k_proj", "v_proj", "o_proj",   # Attention projections
        "gate_proj", "up_proj", "down_proj",       # MLP (FFN) layers
    ],
    lora_alpha                = LORA_RANK * 2,      # alpha = 2× rank → faster convergence
    use_gradient_checkpointing = "unsloth",
    random_state               = 3407,
)

trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
total     = sum(p.numel() for p in model.parameters())
print(f"\n✅ Model ready!")
print(f"   Trainable: {trainable:,} params ({100*trainable/total:.2f}%)")
print(f"   VRAM used: {torch.cuda.memory_allocated()/1e9:.2f} GB")
```

## Section 4 — Understanding the DeepSeek-R1 Chat Template

DeepSeek-R1 models use special tokens to delimit the **internal reasoning chain** from the **final answer**. The GRPOTrainer automatically prepends the `<think>` opening token to every completion, so the model always starts in reasoning mode.

### Token Discovery

We programmatically find the special tokens rather than hardcoding them, making this code robust across model variants:

```python
# Auto-discover the special reasoning tokens from the tokenizer vocabulary
reasoning_start = None
reasoning_end   = None
user_token      = None
assistant_token = None

for token in tokenizer.get_added_vocab().keys():
    if "think" in token and "/" in token:
        reasoning_end   = token   # e.g. </think>
    elif "think" in token:
        reasoning_start = token   # e.g. <think>
    elif "user" in token:
        user_token      = token
    elif "assistant" in token:
        assistant_token = token

print(f"Reasoning start token: {repr(reasoning_start)}")
print(f"Reasoning end token:   {repr(reasoning_end)}")
print(f"User token:            {repr(user_token)}")
print(f"Assistant token:       {repr(assistant_token)}")

assert reasoning_start and reasoning_end, "❌ Could not find <think> tokens — check model/tokenizer!"
print("\n✅ All special tokens found!")
```

### System Prompt for Language Targeting

We instruct the model to reason in **Bahasa Indonesia** via the system prompt. The language-consistency reward function (Section 5) then reinforces this behaviour during training.

{% hint style="info" %}
Design principle: the system prompt _asks_ for the target language; the reward function _enforces_ it. Neither alone is sufficient — both are needed for reliable steering.
{% endhint %}

```python
# System prompt that instructs the model to reason in Bahasa Indonesia
system_prompt = (
    "You are given a problem.\n"
    "Think about the problem and provide your working out.\n"
    "You must think in Bahasa Indonesia."
)

print("System prompt:")
print("-" * 50)
print(system_prompt)
print("-" * 50)
```

```python
# Visualise how the chat template formats a conversation
# This shows exactly what the model "sees" as input tokens

example_conversation = [
    {"role": "user",      "content": "What is 1+1?"},
    {"role": "assistant", "content": f"<think>I think it's 2.</think>2"},
    {"role": "user",      "content": "What is 3+3?"},
    {"role": "assistant", "content": f"<think>That would be 6.</think>6"},
]

formatted = tokenizer.apply_chat_template(
    example_conversation,
    tokenize              = False,
    add_generation_prompt = True,
)

print("Formatted conversation (what the model sees):")
print("=" * 60)
print(formatted)
print("=" * 60)
print("\n👆 Notice the <think> / </think> tags wrapping the reasoning.")
print("   GRPOTrainer prepends <think> automatically — the model must close it and then give the answer.")
```

## Section 5 — Prepare the DAPO-Math Dataset

We use [DAPO-Math-17k](https://huggingface.co/datasets/open-r1/DAPO-Math-17k-Processed) — a curated math reasoning dataset from the Open-R1 project, containing 17k problems with ground-truth solutions compatible with GRPO training.

### Why Filter by Token Length?

GRPO requires every prompt in a batch to fit within `max_prompt_length`. Rather than truncating (which corrupts problems), we **remove the top 10% longest prompts**. This ensures:

* No silent truncation of math problem statements
* Predictable memory usage per step
* Faster per-step time (no padding to extreme lengths)

### Data Pipeline

```
Raw DAPO-Math (17k examples)
        │
        ▼  apply_chat_template → count tokens
        │
        ▼  filter: keep bottom 90% by token length
        │
        ▼  format: add system prompt, extract answer field
        │
   Train Dataset (~15k examples, ready for GRPOTrainer)
```

```python
from datasets import load_dataset

print("Loading DAPO-Math-17k (English split) ...")
dataset = load_dataset("open-r1/DAPO-Math-17k-Processed", "en", split="train")
print(f"Raw dataset: {len(dataset)} examples")
print(f"Columns: {dataset.column_names}")
```

```python
# Inspect a raw example before formatting
print("=== Raw prompt (first example) ===")
print(dataset[0]["prompt"])
print()
print("=== Raw solution (first example) ===")
print(dataset[0]["solution"][:300], "...")
```

```python
# Answer extraction helper
# For DAPO-Math the solution IS the answer — no #### parsing needed.
# (For GSM8K you would use: text.split("####")[1].strip())

def extract_answer(text):
    """
    Extract the answer field from a solution string.
    DAPO-Math solutions are already clean — we return them as-is.
    For GSM8K, uncomment the #### split logic.
    """
    # GSM8K style (uncomment if switching datasets):
    # if "####" in text:
    #     return text.split("####")[1].strip()
    return text

# Format the full dataset with system prompt and answer extraction
dataset = dataset.map(lambda x: {
    "prompt": [
        {"role": "system", "content": system_prompt},
        {"role": "user",   "content": x["prompt"]},
    ],
    "answer": extract_answer(x["solution"]),
})

print(f"\nFormatted dataset: {len(dataset)} examples")
print("\nFirst example prompt structure:")
import json
print(json.dumps(dataset[0]["prompt"], indent=2))
```

```python
# Filter out the top 10% longest prompts to avoid truncation
# This is critical — truncated math problems make training noise, not signal.

import numpy as np

print("Tokenising prompts to measure lengths ...")
tokenized = dataset.map(
    lambda x: {
        "tokens": tokenizer.apply_chat_template(
            x["prompt"],
            add_generation_prompt=True,
            tokenize=True,
        )
    },
    batched=True,
)
tokenized = tokenized.map(lambda x: {"L": len(x["tokens"])})

# Find the 90th percentile cutoff
lengths       = np.array(tokenized["L"])
max_length_90 = int(np.quantile(lengths, 0.9))

print(f"Token length stats:")
print(f"  Min:        {lengths.min()}")
print(f"  Median:     {int(np.median(lengths))}")
print(f"  90th pct:   {max_length_90}  ← cutoff")
print(f"  Max:        {lengths.max()}")

# Apply the filter
keep_indices = np.where(lengths <= max_length_90)[0]
dataset      = dataset.select(keep_indices)
del tokenized

print(f"\n✅ After filtering: {len(dataset)} examples ({len(keep_indices)/len(lengths)*100:.1f}% retained)")
print(f"   Max prompt length for training: {max_length_90 + 1} tokens")
```

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

* **`match_format_exactly`** gives a strong binary signal for correct structure
* **`match_format_approximately`** provides gradient even for near-miss formatting
* **`check_answer`** rewards exact string matches and ratio-based proximity
* **`check_numbers`** catches cases where the answer is embedded in a sentence
* **`language_consistency`** is the novel reward that steers the reasoning language

Using multiple rewards is better than one composite reward because each signal is cleaner and easier to tune independently.

```python
import re

# Regex that matches everything after the closing </think> token
# This captures the model's final answer (after the reasoning chain ends)
solution_end_regex = rf"{reasoning_end}(.*)"
match_format       = re.compile(solution_end_regex, re.DOTALL)

# Number extraction regex — handles negatives, decimals, and commas (e.g. 123,456)
match_numbers = re.compile(
    r".*?[\s]{0,}([-]?[\d\.\,]{1,})",
    flags=re.MULTILINE | re.DOTALL,
)

# ── Quick unit tests ──────────────────────────────────────────────────────────
print("Testing match_format regex:")
print(f"  Valid:   {match_format.findall('<think>steps</think>The answer is 4.')}")
print(f"  No tag:  {match_format.findall('The answer is 4.')}")

print("\nTesting match_numbers regex:")
print(f"  '0.34':    {match_numbers.findall('  0.34  ')}")
print(f"  '123,456': {match_numbers.findall('  123,456  ')}")
print(f"  '-0.234':  {match_numbers.findall('  -0.234  ')}")
print(f"  '17':      {match_numbers.findall('17')}")
```

```python
# ── Reward 1: Exact format match ─────────────────────────────────────────────

def match_format_exactly(completions, **kwargs):
    """
    +3.0 if the completion contains a valid </think>...answer structure.
    
    This is the primary structural reward — it fires only when the model
    correctly closes the reasoning block and transitions to the answer.
    """
    scores = []
    for completion in completions:
        response = completion[0]["content"]
        score = 3.0 if match_format.search(response) is not None else 0.0
        scores.append(score)
    return scores


# ── Reward 2: Approximate format match ───────────────────────────────────────

def match_format_approximately(completions, **kwargs):
    """
    Partial credit for tag structure: +0.5 per correct tag count.
    Penalises repeated tags (-1.0 each) which indicates degenerate looping.
    
    Note: We don't reward <think> since GRPOTrainer always prepends it.
    We only score </think> — the model must learn to close the tag.
    """
    scores = []
    for completion in completions:
        response = completion[0]["content"]
        score = 0.0
        # <think> is prepended automatically — penalise if the model adds another
        score += 0.5 if response.count(reasoning_start) == 1 else -1.0
        # </think> must appear exactly once
        score += 0.5 if response.count(reasoning_end)   == 1 else -1.0
        scores.append(score)
    return scores


print("✅ Structural reward functions defined!")
print()
# Unit tests
test_good  = f"<think>working...</think>42"
test_multi = f"<think>step1</think><think>step2</think>42"
test_none  = "The answer is 42."
print(f"  Good response:   format_exactly={match_format_exactly([[{'content': test_good}]])}, "
      f"format_approx={match_format_approximately([[{'content': test_good}]])}")
print(f"  Multi-think:     format_exactly={match_format_exactly([[{'content': test_multi}]])}, "
      f"format_approx={match_format_approximately([[{'content': test_multi}]])}")
print(f"  No tags:         format_exactly={match_format_exactly([[{'content': test_none}]])}, "
      f"format_approx={match_format_approximately([[{'content': test_none}]])}")
```

```python
# ── Reward 3: Answer correctness (string + ratio-based) ──────────────────────

def check_answer(prompts, completions, answer, **kwargs):
    """
    Reward based on how close the extracted answer is to the ground truth.
    
    Scoring tiers:
      +5.0  Exact string match
      +3.5  Match after stripping whitespace
      +2.0  Numeric ratio within 10% (0.9 ≤ guess/true ≤ 1.1)
      +1.5  Numeric ratio within 20%
      -2.5  Wrong numeric answer (parseable but incorrect)
      -4.5  Answer not parseable as float
      -2.0  No </think> found — cannot extract answer
    
    The ratio-based rewards help during early training when the model is
    learning to produce the right magnitude, even if not perfectly precise.
    """
    responses = [completion[0]["content"] for completion in completions]
    extracted = [
        m.group(1) if (m := match_format.search(r)) is not None else None
        for r in responses
    ]

    scores = []
    for guess, true_answer in zip(extracted, answer):
        if guess is None:
            scores.append(-2.0)
            continue
        if guess == true_answer:
            scores.append(5.0)
        elif guess.strip() == true_answer.strip():
            scores.append(3.5)
        else:
            try:
                ratio = float(guess) / float(true_answer)
                if   0.9 <= ratio <= 1.1: scores.append(2.0)
                elif 0.8 <= ratio <= 1.2: scores.append(1.5)
                else:                     scores.append(-2.5)
            except:
                scores.append(-4.5)
    return scores


print("✅ check_answer defined!")
```

```python
# ── Reward 4: Numeric extraction + float comparison ──────────────────────────

PRINTED_TIMES     = 0
PRINT_EVERY_STEPS = 5    # Print a sample every N steps to monitor training progress

def check_numbers(prompts, completions, answer, **kwargs):
    """
    Float-based answer comparison using a more aggressive number extraction.
    
    Complements check_answer by handling cases where the answer is embedded
    in a sentence: 'The result is $20.' → extracts 20.0
    
    Also removes commas: '1,234' → 1234.0
    
    Prints a training sample every PRINT_EVERY_STEPS for monitoring.
    """
    global PRINTED_TIMES

    question  = prompts[0][-1]["content"]
    responses = [completion[0]["content"] for completion in completions]
    extracted = [
        m.group(1) if (m := match_numbers.search(r)) is not None else None
        for r in responses
    ]

    if PRINTED_TIMES % PRINT_EVERY_STEPS == 0:
        print("*" * 20 + f" Step sample (printed every {PRINT_EVERY_STEPS} steps)")
        print(f"Question:  {question[:120]}...")
        print(f"Answer:    {answer[0]}")
        print(f"Response:  {responses[0][:200]}...")
        print(f"Extracted: {extracted[0]}")
        print()
    PRINTED_TIMES += 1

    scores = []
    for guess, true_answer in zip(extracted, answer):
        if guess is None:
            scores.append(-2.5)
            continue
        try:
            true_f = float(true_answer.strip())
            guess_f = float(guess.strip().replace(",", ""))
            scores.append(3.5 if guess_f == true_f else -1.5)
        except:
            scores.append(0.0)   # non-numeric answer — neutral score
    return scores


print("✅ check_numbers defined!")
```

```python
# ── Reward 5: Language consistency (Bahasa Indonesia) ────────────────────────
# Inspired by DeepSeek-R1 paper Section 3.3: "Language Consistency Reward"

import langid

def get_lang(text: str) -> str:
    """Detect the language of a text string. Returns ISO 639-1 code or 'und' for empty."""
    if not text:
        return "und"
    lang, _ = langid.classify(text)
    return lang

# Quick test
print("Language detection sanity checks:")
print(f"  English:    '{get_lang('Hello, how are you?')}' (expected: en)")
print(f"  Indonesian: '{get_lang('Aku berpikir kalau ini adalah jawaban yang benar')}' (expected: id)")
print(f"  Chinese:    '{get_lang('我在这里')}' (expected: zh)")
```

```python
def format_and_language_reward_func(completions, **kwargs):
    """
    Reward the model for generating reasoning traces in Bahasa Indonesia.
    
    Scoring:
      +5.0  Detected as Indonesian ('id') — target language achieved!
      -3.0  Detected as English ('en')   — model defaulted to English
      -3.0  Detected as Chinese ('zh')   — base model leaks Chinese
      -5.0  Any other language or malformed completion
    
    This directly implements the 'language consistency reward' from the
    DeepSeek-R1 paper, which was used to prevent language mixing in
    the base model's reasoning chains.
    """
    scores = []
    for completion_item in completions:
        # Validate completion structure
        if (not completion_item or
                not isinstance(completion_item[0], dict) or
                "content" not in completion_item[0]):
            scores.append(-5.0)
            print(f"⚠️  Malformed completion, assigning -5.0: {completion_item}")
            continue

        content = completion_item[0]["content"]
        lang    = get_lang(content)

        if   lang == "id": scores.append( 5.0)
        elif lang == "en": scores.append(-3.0)
        elif lang == "zh": scores.append(-3.0)
        else:              scores.append(-5.0)

    return scores


# ── Unit test ─────────────────────────────────────────────────────────────────
test_completions = [
    [{"role": "assistant", "content": "Saya berpikir bahwa jawabannya adalah 12 karena 3 × 4 = 12."}],  # Indonesian
    [{"role": "assistant", "content": "I think the answer is 12 because 3 × 4 = 12."}],                 # English
    [{"role": "assistant", "content": "我认为答案是12，因为3×4=12。"}],                                    # Chinese
]
results = format_and_language_reward_func(completions=test_completions)
print("Language reward test:")
print(f"  Indonesian: {results[0]}  (expected  5.0)")
print(f"  English:    {results[1]}  (expected -3.0)")
print(f"  Chinese:    {results[2]}  (expected -3.0)")
```

## Section 7 — Configure & Launch GRPO Training

### vLLM Integration

When `fast_inference=True`, GRPOTrainer routes all rollout generation through vLLM's `SamplingParams`. This gives a significant speedup on H100:

| Method                        | 4 rollouts time (approx) |
| ----------------------------- | ------------------------ |
| Standard HuggingFace generate | \~8 seconds              |
| vLLM with PagedAttention      | \~2 seconds              |

We configure `SamplingParams` separately from `GRPOConfig` — vLLM uses its own sampling API.

### Sequence Length Calculation

We compute `max_completion_length` dynamically from the measured max prompt length, ensuring the total sequence always fits within `MAX_SEQ_LENGTH`:

```
max_completion_length = MAX_SEQ_LENGTH - max_prompt_length - 1
```

This prevents silent truncation of reasoning chains, which would produce noisy gradients.

### Training Duration

We run for **100 steps** here (`max_steps=100`) — enough to see rewards start climbing, and to verify the training loop works. For production quality, train for 1–3 full epochs (`num_train_epochs=1`, remove `max_steps`).

{% hint style="info" %}
Watch the `reward` and `rewards/language` columns in the training table. You expect `reward` to climb from \~0 to \~2+ after 100 steps, and the language reward to become increasingly positive as the model adopts Indonesian.
{% endhint %}

```python
from vllm import SamplingParams
from trl  import GRPOConfig, GRPOTrainer
import os

# ── Compute dynamic sequence lengths ──────────────────────────────────────────
max_prompt_length     = max_length_90 + 1              # +1 buffer
max_completion_length = MAX_SEQ_LENGTH - max_prompt_length

print(f"Sequence length budget:")
print(f"  MAX_SEQ_LENGTH:        {MA_SEQ_LENGTH}")
print(f"  max_prompt_length:     {max_prompt_length}")
print(f"  max_completion_length: {max_completion_length}")

# ── vLLM sampling parameters (used for GRPO rollouts) ─────────────────────────
vllm_sampling_params = SamplingParams(
    min_p                    = 0.1,              # Nucleus sampling floor — prevents degenerate tokens
    top_p                    = 1.0,
    top_k                    = -1,               # Disabled (use min_p instead)
    seed                     = 3407,
    stop                     = [tokenizer.eos_token],
    include_stop_str_in_output = True,           # Keep EOS in output for clean parsing
)

# ── GRPO training configuration ───────────────────────────────────────────────
OUTPUT_DIR = "/workspace/outputs"
os.makedirs(OUTPUT_DIR, exist_ok=True)

training_args = GRPOConfig(
    # vLLM integration
    vllm_sampling_params       = vllm_sampling_params,
    temperature                = 1.0,            # High temp for diverse rollout exploration

    # Optimiser
    learning_rate              = 5e-6,
    weight_decay               = 0.001,
    warmup_ratio               = 0.1,
    lr_scheduler_type          = "linear",
    optim                      = "adamw_8bit",
    max_grad_norm              = 1.0,            # Less aggressive clipping than vision notebook

    # Batch / rollout
    per_device_train_batch_size = 1,
    gradient_accumulation_steps = 1,             # Increase to 4 for smoother loss curves
    num_generations             = 4,             # vLLM makes this fast even at 4

    # Sequence lengths
    max_prompt_length           = max_prompt_length,
    max_completion_length       = max_completion_length,

    # Duration — 100 steps for validation; use num_train_epochs=1 for production
    max_steps   = 100,
    save_steps  = 100,
    logging_steps = 1,

    # Logging
    report_to  = "none",     # Change to "wandb" to enable Weights & Biases
    output_dir = OUTPUT_DIR,
)

print(f"\n✅ Training configuration ready!")
print(f"   Steps:           {training_args.max_steps}")
print(f"   Generations:     {training_args.num_generations}")
print(f"   Effective batch: {training_args.per_device_train_batch_size * training_args.gradient_accumulation_steps}")
```

```python
# Build the GRPOTrainer with all five reward functions
# Reward functions are applied in order; their scores are summed.

trainer = GRPOTrainer(
    model            = model,
    processing_class = tokenizer,
    reward_funcs     = [
        match_format_exactly,           # +3.0   — binary structural signal
        match_format_approximately,     # +1.0   — gradient for near-miss format
        check_answer,                    # +5.0   — string + ratio correctness
        check_numbers,                   # +3.5   — float extraction correctness
        format_and_language_reward_func, # +5.0 — Bahasa Indonesia compliance
    ],
    args             = training_args,
    train_dataset    = dataset,
)

print("✅ GRPOTrainer initialised. Launching training ...")
print()
print("What to watch in the training table:")
print("  reward           — total reward (aim: climbing from ~0 to 2+)")
print("  rewards/format   — should rise quickly in first 20 steps")
print("  rewards/language — Indonesian compliance (may take 50+ steps to rise)")
print("  completion_length — should stabilise as model learns to close </think>")
print()

# ─── LAUNCH TRAINING ──────────────────────────────────────────────────────────
trainer.train()
print("\n🎉 Training complete!")
```

## Section 8 — Inference & Language Compliance Evaluation

We run three comparison experiments:

1. **Without LoRA** — baseline model behaviour (no system prompt)
2. **With LoRA, no system prompt** — does LoRA affect general reasoning?
3. **With LoRA + system prompt** — the intended deployment mode (Indonesian reasoning)

Then we run a **batch evaluation on 20 samples** to measure the language compliance rate quantitatively.

```python
from vllm import SamplingParams

inference_params = SamplingParams(
    temperature = 1.0,
    top_k       = 50,
    max_tokens  = 1024,
)

# ── Experiment 1: Baseline (no LoRA, no system prompt) ────────────────────────
print("=== Experiment 1: Baseline — no LoRA, no system prompt ===")
print("Question: What is the sqrt of 101?")
print("-" * 60)

output = model.fast_generate(
    ["What is the sqrt of 101?"],
    sampling_params = inference_params,
    lora_request    = None,
)[0].outputs[0].text

print(output)
print("-" * 60)
print(f"Detected language: {get_lang(output)}")
```

```python
# Save the LoRA adapters before loading them for inference
# fast_generate() requires save_lora() before load_lora()
print("Saving LoRA adapters ...")
model.save_lora("/workspace/grpo_lora")
print("✅ LoRA saved to /workspace/grpo_lora")
```

```python
# ── Experiment 2: With LoRA, no system prompt ─────────────────────────────────
print("=== Experiment 2: With LoRA — no system prompt ===")
print("Question: Solve (x + 2)^2 = 0")
print("(Testing whether LoRA preserves general math ability)")
print("-" * 60)

messages = [{"role": "user", "content": "Solve (x + 2)^2 = 0"}]
text = tokenizer.apply_chat_template(
    messages, add_generation_prompt=True, tokenize=False
)

output = model.fast_generate(
    text,
    sampling_params = inference_params,
    lora_request    = model.load_lora("/workspace/grpo_lora"),
)[0].outputs[0].text

print(output)
print("-" * 60)
print(f"Detected language: {get_lang(output)}")
```

```python
# ── Experiment 3: With LoRA + system prompt (target deployment mode) ──────────
print("=== Experiment 3: With LoRA + Indonesian system prompt ===")
print("Question: Solve (x + 2)^2 = 0")
print("(Expected: reasoning in Bahasa Indonesia)")
print("-" * 60)

messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user",   "content": "Solve (x + 2)^2 = 0"},
]
text = tokenizer.apply_chat_template(
    messages, add_generation_prompt=True, tokenize=False
)

output = model.fast_generate(
    text,
    sampling_params = inference_params,
    lora_request    = model.load_lora("/workspace/grpo_lora"),
)[0].outputs[0].text

print(output)
print("-" * 60)
print(f"Detected language: {get_lang(output)}")
```

```python
# ── Experiment 4: Control — system prompt but NO LoRA ─────────────────────────
print("=== Experiment 4: System prompt only — no LoRA (control) ===")
print("(Does the system prompt alone achieve Indonesian reasoning?)")
print("-" * 60)

messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user",   "content": "Solve (x + 2)^2 = 0"},
]
text = tokenizer.apply_chat_template(
    messages, add_generation_prompt=True, tokenize=False
)

output_no_lora = model.fast_generate(
    text,
    sampling_params = inference_params,
    lora_request    = None,
)[0].outputs[0].text

print(output_no_lora)
print("-" * 60)
print(f"Detected language: {get_lang(output_no_lora)}")
print()
print("Compare Experiments 3 vs 4 — LoRA should show higher Indonesian compliance.")
```

### Batch Language Compliance Evaluation

A single example is anecdotal. We run over **20 randomly sampled problems** and measure the percentage that produce Indonesian reasoning chains — with and without the LoRA.

```python
# Batch evaluation: Indonesian language compliance across 20 samples
sample_dataset = dataset.shuffle(seed=3407).select(range(20))

with_lora_id    = 0
without_lora_id = 0

print("Running language compliance evaluation on 20 samples ...")
print("(This takes ~1–2 minutes with vLLM)")
print("=" * 60)

eval_params = SamplingParams(temperature=1.0, top_k=50, max_tokens=512)

for i, sample in enumerate(sample_dataset):
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user",   "content": sample["prompt"][1]["content"]},
    ]
    text = tokenizer.apply_chat_template(
        messages, add_generation_prompt=True, tokenize=False
    )

    # With LoRA
    out_lora = model.fast_generate(
        text, sampling_params=eval_params,
        lora_request=model.load_lora("/workspace/grpo_lora"),
    )[0].outputs[0].text

    # Without LoRA
    out_base = model.fast_generate(
        text, sampling_params=eval_params,
        lora_request=None,
    )[0].outputs[0].text

    lang_lora = get_lang(out_lora)
    lang_base = get_lang(out_base)

    if lang_lora == "id": with_lora_id    += 1
    if lang_base == "id": without_lora_id += 1

    if (i + 1) % 5 == 0:
        print(f"  Processed {i+1}/20  |  LoRA Indonesian so far: {with_lora_id}  |  Base: {without_lora_id}")

print()
print("=" * 60)
print("LANGUAGE COMPLIANCE RESULTS (20 samples)")
print("=" * 60)
print(f"  With LoRA:    {with_lora_id}/20 Indonesian  ({with_lora_id/20*100:.1f}%)")
print(f"  Without LoRA: {without_lora_id}/20 Indonesian ({without_lora_id/20*100:.1f}%)")
print(f"  Improvement:  +{with_lora_id - without_lora_id} responses with LoRA")
print()
if with_lora_id > without_lora_id:
    print("✅ LoRA successfully improved Indonesian language compliance!")
else:
    print("ℹ️  Language compliance similar — try training for more steps (500+) for stronger effect.")
```

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
import os
LORA_SAVE_DIR = "/workspace/grpo_lora"
os.makedirs(LORA_SAVE_DIR, exist_ok=True)

# ── Option 1: Save LoRA adapters (already done above, confirmed here) ─────────
print("LoRA adapters location:")
import subprocess
result = subprocess.run(f"ls -lh {LORA_SAVE_DIR}", shell=True, capture_output=True, text=True)
print(result.stdout)
```

```python
# ── Option 2: Push LoRA to HuggingFace Hub ────────────────────────────────────
# Uncomment and fill in your credentials to publish.

# import os
# HF_TOKEN = os.environ.get("HF_TOKEN", "YOUR_TOKEN_HERE")
# model.push_to_hub("your_username/deepseek-r1-qwen3-8b-indonesian-grpo", token=HF_TOKEN)
# tokenizer.push_to_hub("your_username/deepseek-r1-qwen3-8b-indonesian-grpo", token=HF_TOKEN)

# ── Option 3: Merge to 16-bit float (for vLLM standalone deployment) ──────────
# if False:
#     model.save_pretrained_merged("/workspace/model_16bit", tokenizer, save_method="merged_16bit")
#     print("✅ Merged 16-bit model saved to /workspace/model_16bit")

# ── Option 4: Merge to 4-bit (smaller deployment) ────────────────────────────
# if False:
#     model.save_pretrained_merged("/workspace/model_4bit", tokenizer, save_method="merged_4bit")

# ── Option 5: Export to GGUF (llama.cpp / Ollama) ────────────────────────────
# if False:
#     model.save_pretrained_gguf("/workspace/model_gguf", tokenizer, quantization_method="q4_k_m")
#     # Other options: "q8_0", "f16", "q5_k_m"

print("Uncomment the export option you need and re-run this cell.")
print()
print("To copy adapters to your local machine:")
print("  scp -r root@<POD_IP>:/workspace/grpo_lora ./grpo_lora_local")
```
