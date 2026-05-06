---
icon: compress
---

# Quantization

## Overview

Quatization aims at transforming high-precision large models (such as FLUX.1-dev) from Hugging Face into a compressed INT4 format using the SVDQuant algorithm.

**A Simple Example:**

Suppose you have data ranging from 0 to 100, and you want to represent it using **8-bit integers** (range 0-255):

* Original: 3.14159 → Quantized to 8-bit: 3
* Original: 27.82 → Quantized to 8-bit: 28
* Original: 99.9 → Quantized to 8-bit: 100

**The formula:**

```
quantized_value = original_value / scale_factor
scale_factor = original_max / quantizer_max = 100 / 255 ≈ 0.39
```

Memory usage drops from 32-bit float (4 bytes) to 8-bit integer (1 byte)—**75% compression**. Simple and effective.

## Configuration Parameters

* **Model source:** Use models hosted on Hugging Face Hub/ Model Scope .
* **URL:** The full link to the model repository
  * _Example_: `https://huggingface.co/black-forest-labs/FLUX.1-dev`
  * :exclamation:Currently, only models based on the FLUX-1.dev architecture are supported
* **Access Token:** Your personal Hugging Face User Access Token (with `Read` permission).
  * get access to them from hugging face hub/model scope account settings.
* **Architecture:** The target model architecture for quantization (FLUX.1-dev).
* **Prompt:** Example input texts or prompts that can be used to calibrate or test the model during quantization.
  * Helps the quantization algorithm maintain accuracy on common inputs.
  * Filled in automatically for you.
* **Precision：**&#x54;he target numerical precision for the model.
  * **INT4 (4-bit Integer):** Uses a uniform grid. It divides the dynamic range into 16 equal steps. This linear approach often struggles with the "non-uniform distribution"(long-tail distribution) typically found in deep learning models.It offers **excellent universal compatibility**, supported by almost all modern NVIDIA RTX GPUs.It makes total sense to optimize your model around INT4, where the ecosystem is currently centered at.
  * **NVFP4 (4-bit Floating Point):** Uses an exponential distribution, typically structured with 1 sign bit, 2 exponent bits, and 1 mantissa bit (E2M1). This structure provides higher precision for values near zero (where most weights are concentrated) while using the exponent bits to better capture "outliers." It is a flagship feature of **the NVIDIA Blackwell architecture**.Once Blackwell reaches critical mass globally, FP4-native models will dominate too.
* **Algorithm:** The method used for quantization. **SVDQUANT** is automatically selected.Check the links for more information on SVDQUANT :arrow\_down:
  * [Website](https://hanlab.mit.edu/projects/svdquant)
  * [Github](https://github.com/nunchaku-ai/nunchaku)
  * [Paper](https://arxiv.org/abs/2411.05007)
* **Output Model:** output model name.

## Deploy

* Click the deploy button on the right side of the page.

<figure><img src="../.gitbook/assets/image (215).png" alt=""><figcaption></figcaption></figure>

* Check your in-progress/history tasks here.

<figure><img src="../.gitbook/assets/image (216).png" alt=""><figcaption></figcaption></figure>



