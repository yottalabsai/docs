---
icon: dollar-sign
---

# Pricing

Each user will have $xxx of free quota every day. Once the quota is exceeded, we will deduct from your balance.

## Text Generation Model Pricing

* **Llama 3.1 8B**: $0.08 per 1M tokens
* **Llama 3.2 3B**: $0.04 per 1M tokens
* **Mistral 7B**: $0.20 per 1M tokens

## Image Generation Model Pricing

The price for generating an image depends on both the **size** of the image and the **step** parameter used. The following price structure assumes that the image will be **1024x1024 pixels** (about **1M pixels**) and that **25 steps** will be used. The actual charge for your image generation will vary based on the **image size** and the number of **steps**.

**Base Rate for Image Generation:**

* **FLUX.1 dev**: $0.025 per 1M Pixels @ 25 Steps
* **FLUX.1 Schnell**: $0.018 per 1M Pixels @ 25 Steps
* **FLUX.1 Schnell Int4**: $0.008 per 1M Pixels @ 25 Steps
* **SANA**: $0.004 per 1M Pixels @ 25 Steps
* **SANA Int4**: $0.001 per 1M Pixels @ 25 Steps

**Pricing Formula:**

The cost is determined by multiplying the **Base Rate** by the **adjusted size** of the image and the **number of steps** used.

* **Image Size**: The cost scales with the width and height of the image. For example:
  * For **512x512 pixels** (a quarter of 1024x1024): The cost would be **Base Rate \* (512/1024) \* (512/1024) \* (steps/25)**.
  * For **2048x2048 pixels** (four times the size of 1024x1024): The cost would be **Base Rate \* (2048/1024) \* (2048/1024) \* (steps/25)**.
* **Step Adjustment**: The price also scales based on the number of steps:
  * **25 steps**: No adjustment needed (multiplier = 1).
  * **50 steps**: The multiplier is **2** (i.e., steps/25 = 50/25 = 2).

**Example Calculation:**

Let’s say you want to generate an image using the **FLUX.1 dev** model with **512x512 pixels** and **50 steps**:

* The **Base Rate** for **FLUX.1 dev** is **$0.025 per 1M Pixels @ 25 Steps**.
* **Width and height** are both 512, which is half the size of 1024, so the image scaling factor would be **(512/1024) \* (512/1024) = 0.25**.
* **Step adjustment**: Since you're using 50 steps, the step multiplier would be **2** (i.e., steps/25 = 50/25 = 2).

Thus, the total cost would be:

**$0.025 \* 0.25 \* 0.25 \* 2 = $0.003125 per image**.
