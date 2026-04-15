# Fine-tuning Orpheus\_(3B)-TTS with Unsloth

> :sloth:This tutorial is created based on [Unsloth official notebooks](https://unsloth.ai/docs/get-started/unsloth-notebooks).&#x20;

Welcome to the Orpheus text-to-speech model fine-tuning tutorial. I'll walk you through the entire process step by step, from zero to having your own custom-trained TTS model up and running.

### What is a TTS model?

A TTS (Text-to-Speech) model converts written text into spoken audio. It reads text and generates a voice that sounds natural and human-like.

### What You'll Need

First things first - this tutorial is designed to run on pods with our official template of unsloth. Create one and get started!

{% stepper %}
{% step %}
### Step 1: Installing Dependencies

This part's a bit boring but crucial. We need to install a bunch of libraries. Run this in your `/workspace/notebook.ipynb`:

```python
%%capture
# Install Unsloth and core dependencies
%pip install unsloth

# These specific versions are important - don't change them
%pip install transformers==4.56.2
%pip install --no-deps trl==0.22.2
%pip install snac torchcodec "datasets>=3.4.1,<4.0.0"
```

**Note**: If you encounter any CUDA version mismatches or xformers errors, you might need to install a specific xformers version matching your PyTorch installation. Check your PyTorch version with `torch.__version__` and install the corresponding xformers.

{% hint style="info" %}
After installation, restart your kernel (Kernel → Restart Kernel in Jupyter).
{% endhint %}
{% endstep %}

{% step %}
### Loading the Model

We're using the Unsloth framework, which is awesome because it's fast and memory-efficient. The base model is `orpheus-3b-0.1-ft`, which has about 3 billion parameters:

```python
from unsloth import FastLanguageModel
import torch

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name = "unsloth/orpheus-3b-0.1-ft",
    max_seq_length = 2048,  # Sequence length - leave this as is
    dtype = None,  # Auto-detect precision
    load_in_4bit = False,  # Set to True if you're low on VRAM
)
```

You'll see something like this :arrow\_down: This indicates that unsloth has successfully started and loaded the model's safetensors.

<figure><img src="../../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>
{% endstep %}

{% step %}
### Setting Up LoRA

LoRA is a lifesaver - it lets us train only a small portion (1-10%) of the model's parameters, drastically reducing training costs:

```python
model = FastLanguageModel.get_peft_model(
    model,
    r = 64,  # LoRA rank - higher = better quality but slower
    target_modules = ["q_proj", "k_proj", "v_proj", "o_proj",
                      "gate_proj", "up_proj", "down_proj"],
    lora_alpha = 64,
    lora_dropout = 0,
    bias = "none",
    use_gradient_checkpointing = "unsloth",  # Unsloth's magic - saves 30% VRAM!
    random_state = 3407,
)
```

* `r = 64`: This is adjustable - try 8/16/32/64/128. Higher values train slower but theoretically give better results
* `lora_alpha`: Usually best to keep this equal to `r`
* `use_gradient_checkpointing = "unsloth"`: This is Unsloth's secret sauce - always use it!

You'll see this after running the cell above:

```python
##Unsloth 2026.2.1 patched 28 layers with 28 QKV layers, 28 O layers and 28 MLP layers.
```
{% endstep %}

{% step %}
### Preparing Your Data (Important!)

This is the most complex part of the whole pipeline, and where things most often go wrong. We need to convert audio into tokens the model can understand.

#### :checkered\_flag:Data Format Requirements

* **Single speaker**: You need `text` and `audio` fields
* **Multi-speaker**: You need `source` (speaker name), `text`, and `audio` fields

This example uses the `MrDragonFox/Elise` dataset, which is single-speaker. If you're using your own dataset, make sure the format matches:

```python
from datasets import load_dataset

# Load the dataset
dataset = load_dataset("MrDragonFox/Elise", split="train")
# Or load from local files:
# dataset = load_dataset("json", data_files="your_data.json", split="train")
```

#### :checkered\_flag:Audio Encoding (SNAC)

We now use the SNAC model to convert audio waveforms into discrete tokens. Don't let the code volume intimidate you - the concept is simple:

1. Resample audio to 24kHz
2. Use SNAC to encode into multi-layer codes
3. Interleave these codes in a specific order

```python
import torchaudio.transforms as T
from snac import SNAC

# Load the SNAC model
snac_model = SNAC.from_pretrained("hubertsiuzdak/snac_24khz").to("cuda")

def tokenise_audio(waveform):
    """Convert audio waveform to tokens"""
    waveform = torch.from_numpy(waveform).unsqueeze(0)
    waveform = waveform.to(dtype=torch.float32)
    
    # Resample to 24kHz
    resample_transform = T.Resample(orig_freq=ds_sample_rate, new_freq=24000)
    waveform = resample_transform(waveform).unsqueeze(0).to("cuda")
    
    # Encode
    with torch.inference_mode():
        codes = snac_model.encode(waveform)
    
    # Interleave codes in the correct order (this order matters!)
    all_codes = []
    for i in range(codes[0].shape[1]):
        all_codes.append(codes[0][0][i].item() + 128266)
        all_codes.append(codes[1][0][2*i].item() + 128266 + 4096)
        all_codes.append(codes[2][0][4*i].item() + 128266 + (2*4096))
        all_codes.append(codes[2][0][(4*i)+1].item() + 128266 + (3*4096))
        all_codes.append(codes[1][0][(2*i)+1].item() + 128266 + (4*4096))
        all_codes.append(codes[2][0][(4*i)+2].item() + 128266 + (5*4096))
        all_codes.append(codes[2][0][(4*i)+3].item() + 128266 + (6*4096))
    
    return all_codes

def add_codes(example):
    codes_list = None
    try:
        answer_audio = example.get("audio")
        if answer_audio and "array" in answer_audio:
            audio_array = answer_audio["array"]
            codes_list = tokenise_audio(audio_array)
    except Exception as e:
        print(f"Skipping row due to error: {e}")
    
    example["codes_list"] = codes_list
    return example
```

Now apply this to your entire dataset:

{% hint style="info" %}
Use `%pip install librosa` and `%pip install soundfile` if you cannot import them.
{% endhint %}

```python
ds_sample_rate = dataset[0]["audio"]["sampling_rate"]
dataset = dataset.map(add_codes, remove_columns=["audio"])

# Filter out any failed conversions
dataset = dataset.filter(lambda x: x["codes_list"] is not None)
dataset = dataset.filter(lambda x: len(x["codes_list"]) > 0)
```

#### :checkered\_flag:Removing Duplicate Frames

This is a small optimization - removing consecutive duplicate audio frames speeds up training:

```python
def remove_duplicate_frames(example):
    """Remove consecutive duplicate audio frames"""
    vals = example["codes_list"]
    if len(vals) % 7 != 0:
        raise ValueError("Input list length must be divisible by 7")
    
    result = vals[:7]
    for i in range(7, len(vals), 7):
        if vals[i] != result[-7]:
            result.extend(vals[i:i+7])
    
    example["codes_list"] = result
    return example

dataset = dataset.map(remove_duplicate_frames)
```

#### :checkered\_flag:Building Input Sequences

Final step - combine text and audio tokens into the format the model expects:

```python
# Define special tokens
tokeniser_length = 128256
start_of_text = 128000
end_of_text = 128009
start_of_speech = tokeniser_length + 1
end_of_speech = tokeniser_length + 2
start_of_human = tokeniser_length + 3
end_of_human = tokeniser_length + 4
start_of_ai = tokeniser_length + 5
end_of_ai = tokeniser_length + 6

def create_input_ids(example):
    # Single-speaker model
    text_prompt = example['text']
    # For multi-speaker, use:
    # text_prompt = f"{example['source']}: {example['text']}"
    
    text_ids = tokenizer.encode(text_prompt, add_special_tokens=True)
    text_ids.append(end_of_text)
    
    # Assemble the full sequence: [human start] [text] [human end] [ai start] [audio] [ai end]
    input_ids = (
        [start_of_human] +
        text_ids +
        [end_of_human] +
        [start_of_ai] +
        [start_of_speech] +
        example["codes_list"] +
        [end_of_speech] +
        [end_of_ai]
    )
    
    example["input_ids"] = input_ids
    example["labels"] = input_ids
    example["attention_mask"] = [1] * len(input_ids)
    
    return example

dataset = dataset.map(create_input_ids, remove_columns=["text", "codes_list"])
```

If you're training a multi-speaker model (like the original orpheus), remember to modify the `text_prompt` line to include speaker information!
{% endstep %}

{% step %}
### Training Time

Configure your training parameters:

```python
from transformers import TrainingArguments, Trainer

trainer = Trainer(
    model = model,
    train_dataset = dataset,
    args = TrainingArguments(
        per_device_train_batch_size = 1,  # Batch size - be careful with >1 on multi-GPU
        gradient_accumulation_steps = 4,  # Gradient accumulation - effectively batch=4
        warmup_steps = 5,
        max_steps = 60,  # Quick test - for real training use num_train_epochs=1
        # num_train_epochs = 1,  # Use this for full training
        learning_rate = 2e-4,
        logging_steps = 1,
        optim = "adamw_8bit",  # 8-bit optimizer saves memory
        weight_decay = 0.001,
        lr_scheduler_type = "linear",
        seed = 3407,
        output_dir = "outputs",
    ),
)
```

**Tuning suggestions**:

* `learning_rate`: 2e-4 is pretty stable, but if training is unstable try 1e-4
* `max_steps = 60`: This is just for demo - real training needs hundreds or thousands of steps
* `per_device_train_batch_size`: Try 2 if you have enough VRAM, but stick with 1 for multi-GPU setups

Check if you have enough memory:

```python
gpu_stats = torch.cuda.get_device_properties(0)
start_gpu_memory = round(torch.cuda.max_memory_reserved() / 1024 / 1024 / 1024, 3)
max_memory = round(gpu_stats.total_memory / 1024 / 1024 / 1024, 3)
print(f"GPU = {gpu_stats.name}. Max memory = {max_memory} GB.")
print(f"{start_gpu_memory} GB of memory reserved.")
```

Now let's train:

```python
trainer_stats = trainer.train()
```

Grab a coffee - this will take a while. After training, check out the stats:

<figure><img src="../../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (27).png" alt="" width="139"><figcaption></figcaption></figure>

```python
used_memory = round(torch.cuda.max_memory_reserved() / 1024 / 1024 / 1024, 3)
print(f"Training time: {round(trainer_stats.metrics['train_runtime']/60, 2)} minutes")
print(f"Peak memory usage: {used_memory} GB")
```
{% endstep %}

{% step %}
### Testing Your Model

Time to hear how it sounds!

```python
FastLanguageModel.for_inference(model)  # Switch to inference mode
snac_model.to("cpu")  # Move SNAC to CPU to save VRAM

# Prepare test prompts
prompts = [
    "Hey there my name is Elise, and I'm a speech generation model.",
    "This is a test of the fine-tuned voice synthesis system.",
]

chosen_voice = None  # None for single-speaker, or speaker name for multi-speaker
```

Then run the inference code (it's a bit long because it handles token decoding and audio reconstruction):

```python
# Prepare prompts with speaker prefix if needed
prompts_ = [(f"{chosen_voice}: " + p) if chosen_voice else p for p in prompts]

all_input_ids = []
for prompt in prompts_:
    input_ids = tokenizer(prompt, return_tensors="pt").input_ids
    all_input_ids.append(input_ids)

# Add special tokens
start_token = torch.tensor([[128259]], dtype=torch.int64)  # Start of human
end_tokens = torch.tensor([[128009, 128260]], dtype=torch.int64)  # End of text, End of human

all_modified_input_ids = []
for input_ids in all_input_ids:
    modified_input_ids = torch.cat([start_token, input_ids, end_tokens], dim=1)
    all_modified_input_ids.append(modified_input_ids)

# Pad sequences
max_length = max([modified_input_ids.shape[1] for modified_input_ids in all_modified_input_ids])
all_padded_tensors = []
all_attention_masks = []

for modified_input_ids in all_modified_input_ids:
    padding = max_length - modified_input_ids.shape[1]
    padded_tensor = torch.cat([torch.full((1, padding), 128263, dtype=torch.int64), modified_input_ids], dim=1)
    attention_mask = torch.cat([torch.zeros((1, padding), dtype=torch.int64), torch.ones((1, modified_input_ids.shape[1]), dtype=torch.int64)], dim=1)
    all_padded_tensors.append(padded_tensor)
    all_attention_masks.append(attention_mask)

input_ids = torch.cat(all_padded_tensors, dim=0).to("cuda")
attention_mask = torch.cat(all_attention_masks, dim=0).to("cuda")

# Generate!
generated_ids = model.generate(
    input_ids = input_ids,
    attention_mask = attention_mask,
    max_new_tokens = 1200,
    do_sample = True,
    temperature = 0.6,
    top_p = 0.95,
    repetition_penalty = 1.1,
    num_return_sequences = 1,
    eos_token_id = 128258,
    use_cache = True
)

# Extract audio tokens
token_to_find = 128257
token_to_remove = 128258
token_indices = (generated_ids == token_to_find).nonzero(as_tuple=True)

if len(token_indices[1]) > 0:
    last_occurrence_idx = token_indices[1][-1].item()
    cropped_tensor = generated_ids[:, last_occurrence_idx+1:]
else:
    cropped_tensor = generated_ids

# Process tokens
processed_rows = []
for row in cropped_tensor:
    masked_row = row[row != token_to_remove]
    processed_rows.append(masked_row)

code_lists = []
for row in processed_rows:
    row_length = row.size(0)
    new_length = (row_length // 7) * 7
    trimmed_row = row[:new_length]
    trimmed_row = [t - 128266 for t in trimmed_row]
    code_lists.append(trimmed_row)

# Decode audio
def redistribute_codes(code_list):
    layer_1 = []
    layer_2 = []
    layer_3 = []
    for i in range(len(code_list) // 7):
        layer_1.append(code_list[7*i])
        layer_2.append(code_list[7*i+1] - 4096)
        layer_3.append(code_list[7*i+2] - (2*4096))
        layer_3.append(code_list[7*i+3] - (3*4096))
        layer_2.append(code_list[7*i+4] - (4*4096))
        layer_3.append(code_list[7*i+5] - (5*4096))
        layer_3.append(code_list[7*i+6] - (6*4096))
    
    # Validate and clip codes to valid range (0-4095)
    layer_1 = [max(0, min(4095, x)) for x in layer_1]
    layer_2 = [max(0, min(4095, x)) for x in layer_2]
    layer_3 = [max(0, min(4095, x)) for x in layer_3]
    
    codes = [torch.tensor(layer_1).unsqueeze(0),
             torch.tensor(layer_2).unsqueeze(0),
             torch.tensor(layer_3).unsqueeze(0)]
    
    audio_hat = snac_model.decode(codes)
    return audio_hat

my_samples = []
for code_list in code_lists:
    try:
        samples = redistribute_codes(code_list)
        my_samples.append(samples)
    except Exception as e:
        print(f"Error decoding audio: {e}")
        # Add empty sample as placeholder
        my_samples.append(torch.zeros(1, 1, 24000))

# Save and play the audio!
import scipy.io.wavfile as wavfile
from IPython.display import display, Audio
import os

# Create output directory if it doesn't exist
os.makedirs("generated_audio", exist_ok=True)

for i in range(len(my_samples)):
    print(f"\n{i+1}. {prompts[i]}")
    samples = my_samples[i]
    
    # Convert to numpy array
    audio_array = samples.detach().squeeze().to("cpu").numpy()
    
    # Save as WAV file
    output_path = f"generated_audio/output_{i+1}.wav"
    wavfile.write(output_path, 24000, audio_array)
    print(f"   ✓ Saved to: {output_path}")
    
    # Try to display audio player (works in Jupyter)
    try:
        display(Audio(audio_array, rate=24000))
    except:
        print(f"   → Play the audio file directly: {output_path}")
```

<figure><img src="../../.gitbook/assets/image (29).png" alt="" width="277"><figcaption></figcaption></figure>

How does it sound :hugging: ?If it's not great, you might need to:

* Increase training steps
* Adjust learning rate
* Check your data quality
* Try a larger LoRA rank
{% endstep %}
{% endstepper %}

### :grey\_question:FAQ&#x20;

#### Q: Running out of memory?

A: Try these:

* Enable `load_in_4bit = True`
* Reduce `max_seq_length`
* Lower the LoRA `r` value to 32 or 16
* Decrease batch size (though it's already 1)

#### Q: Training is super slow?

A:

* Make sure you're using GPU, not CPU (run `nvidia-smi` to check)
* Check that your CUDA version matches your PyTorch version
* Consider renting a more powerful GPU pod (RTX 6000 PRO or A100)

#### Q: Results aren't great?

A:

* **Data quality is king!** Make sure audio is clear and text is accurate
* Increase training time (more epochs or steps)
* Check if your learning rate is appropriate
* Ensure your dataset is large enough (at least a few hundred samples)

#### Q: Multi-GPU errors?

A: Set `CUDA_VISIBLE_DEVICES=0` to use only one GPU - this model has issues with multi-GPU setups.

#### Q: Can I train on non-English data?

A: Absolutely! Just make sure your dataset has audio + text in your target language. The tokenizer supports multiple languages.

### Advanced Tips

1. **Data augmentation**: Apply subtle transformations to audio (speed variation, light noise) to increase diversity
2. **Staged training**: Train with higher learning rate first, then fine-tune with lower rate
3. **Mixed precision**: Unsloth already optimizes this automatically - you don't need to worry about it
4. **Monitor training**: Change `report_to` to `"wandb"` or `"tensorboard"` for visual training monitoring

### Happy training! 🎉
