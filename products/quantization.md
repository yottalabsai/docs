---
icon: compress
---

# Quantization

## Overview

Quatization aims at transforming high-precision large models (such as FLUX.1-dev) from Hugging Face into a compressed INT4 format using the SVDQuant algorithm.

## Configuration Parameters

* **Model source:** Use models hosted on Hugging Face Hub/ Model Scope .
* **URL:** The full link to the model repository
  * _Example_: `https://huggingface.co/black-forest-labs/FLUX.1-dev`&#x20;
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
* **Algorithm:** The method used for quantization. **SVDQUANT** is automatically  selected.Check the links for more information on SVDQUANT :arrow\_down:
  * [Website](https://hanlab.mit.edu/projects/svdquant)
  * [Github](https://github.com/nunchaku-ai/nunchaku)
  * [Paper](https://arxiv.org/abs/2411.05007)
* **Output Model:** output model name.

## Deploy

* Click the deploy button on the right side of the page.

<figure><img src="../.gitbook/assets/image (36).png" alt="" width="277"><figcaption></figcaption></figure>

* Check your in-progress/history tasks here.

<figure><img src="../.gitbook/assets/image (37).png" alt="" width="563"><figcaption></figcaption></figure>
