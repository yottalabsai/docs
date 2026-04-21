# Pricing

### 1. Overview

AI Gateway aggregates various types of models, including LLM language models, text-to-image models, and text-to-video models. The billing logic for each type of model has differences.

* LLM models are billed based on the number of tokens consumed, divided into two dimensions: input and output;Some LLM models (such as GLM series) support context caching, with cached tokens billed at a lower cached unit price.

<figure><img src="../../.gitbook/assets/image (21).png" alt="" width="411"><figcaption></figcaption></figure>

* Text-to-image models are generally billed based on the number of images generated; some models have tiered pricing based on resolution, like 1k, 2k or 4k.

<figure><img src="../../.gitbook/assets/image (23).png" alt="" width="329"><figcaption></figcaption></figure>

* Image/Text-to-video models are billed based on the duration (in seconds) of the generated video. Some also has tier pricing based on resolution and existence of audio.

<figure><img src="../../.gitbook/assets/image (25).png" alt="" width="500"><figcaption></figcaption></figure>

### 2. Model Pricing Examples

Below are the reference values for the pricing field of the currently integrated models (prices are in USD, with token-based models priced at USD per million tokens):

| Model             | type  | input | output | cached | Remarks                 |
| ----------------- | ----- | ----- | ------ | ------ | ----------------------- |
| claude sonnet 4.6 | token | $3.00 | $15.00 |        |                         |
| glm 5             | token | $0.95 | $3.04  | $0.19  | Support context caching |
| Seedream 4.5      | image |       | $0.38  |        |                         |

### 3. Common Questions

**Q: Why don't image generation models use token billing?**

The computational resource consumption of image generation models mainly depends on image resolution and generation quantity, with little correlation to the number of prompt text tokens. Therefore, the original providers all charge based on "per image." Some models (such as DALL·E 3) have different pricing for different resolutions, expressed through the resolution\_tiers array.

**Q: Can per\_image and resolution\_tiers coexist?**

No. They are mutually exclusive: use per\_image if all model resolutions have the same price; use resolution\_tiers if different resolutions have different prices. If both exist simultaneously, the data layer should validate and report an error.
