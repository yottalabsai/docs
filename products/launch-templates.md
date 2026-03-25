---
icon: page
---

# Launch Templates

Launch Templates are pre-configured environments designed to get your GPU workloads up and running instantly. Instead of manually configuring images, environment variables, and storage every time, you can select a template to deploy a optimized stack in seconds.

### What are Launch Templates?

A Launch Template acts as a "blueprint" for your Pod. It bundles together:

* A Docker Image: Pre-installed with specific frameworks (e.g., PyTorch, Unsloth).
* Resource Configurations: Optimized default settings for storage and connectivity.
* Environment Setup: Ready-to-use tools like JupyterLab or SSH access.

***

### Official Templates

You can find the official available templates under the Official tab in the Launch Templates library.

* [Pytorch 2.9.0](https://console.yottalabs.ai/compute/templates/34) / [Pytorch 2.8.0](https://console.yottalabs.ai/compute/templates/1)
* [Unsloth](https://console.yottalabs.ai/compute/templates/36)
* [Miles](https://console.yottalabs.ai/compute/templates/35)
* [Skyrl](https://console.yottalabs.ai/compute/templates/37)
* [ComfyUI](https://console.yottalabs.ai/compute/templates/2) / [ComfyUI-nunchaku](https://console.yottalabs.ai/compute/templates/3)
* [Qwen](https://console.yottalabs.ai/compute/templates/5)
* [Wan 2.2](https://console.yottalabs.ai/compute/templates/7) / [Wan2.1](https://console.yottalabs.ai/compute/templates/6)
* [FLUX-1.dev](https://console.yottalabs.ai/compute/templates/4)
* [Crowdcent](https://console.yottalabs.ai/compute/templates/67)

***

### Customization

#### Private Templates

If the official templates don't meet your specific needs, you can switch to the Private tab to access templates you have created or shared within your organization.

#### Create Your Own

Want to standardize your own workflow?

1. Go to the Launch Templates side panel.
2. Click **Create**.
3. Define your custom Docker image, environment variables, and default port mappings. See our doc for[ Custom images](custom-images.md).
4. Save it for one-click deployment in the future.

***

### Accessing Your Templates

1. Choose a template
2. Click **Deploy**
3. You will be navigated to **Pods** page.
4. Start a pod. Follow this [guide](gpu-pods.md) if your need help with how to start a pod.
