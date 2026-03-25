---
icon: docker
---

# Custom Images

### Overview

Yotta provides a collection of **official**, **GPU-optimized base images** designed to help developers quickly build and launch compute environments for AI training, inference, and experimentation. Each image comes preconfigured with essential libraries, tools, and services—including **Python**, **JupyterLab**, and **SSH**—so you can begin working immediately with minimal setup.

You can further customize these images through secondary builds to add your own dependencies, frameworks, or configurations. Alternatively, you may bring your own container image as the base; just ensure that SSH is enabled for Pod access. Both approaches are covered in the sections below.

### Option 1: From Yotta's Official Base Images

#### Base Image Information

**Image name:**

```bash
yottalabsai/pytorch:2.8.0-py3.11-cuda12.8.1-cudnn-devel-ubuntu22.04-2025081902
```

This image is built on top of:

```bash
nvidia/cuda:11.7.1-cudnn8-devel-ubuntu20.04
```

Preinstalled Components:

* Python 3.10
* JupyterLab
* SSH
* Common developer utilities

#### Key Features

* ✅ **GPU-Ready Environment**\
  Preconfigured with CUDA 11.7 and cuDNN 8 to ensure smooth GPU acceleration out of the box.
* ✅ **Preinstalled Python & JupyterLab**\
  Includes Python 3.10, pip, and JupyterLab for immediate notebook-based development.
* ✅ **Integrated SSH**\
  Provides remote shell access for debugging, code editing, file transfers, and automation.
* ✅ **Auto-Start Script**\
  A built-in `/start.sh` script automatically initializes required services (SSH, JupyterLab, etc.) at container startup.

#### Environment Variables

<table><thead><tr><th width="209.56640625">Variable</th><th>Description</th></tr></thead><tbody><tr><td><strong>JUPYTER_PASSWORD</strong></td><td>Sets the login password for Jupyter. If not provided, JupyterLab will fail to start.</td></tr></tbody></table>

{% hint style="warning" %}
**Important:** Ensure you set the required environment variables (especially `JUPYTER_PASSWORD`) before launching the container. Missing values will prevent JupyterLab from initializing correctly.
{% endhint %}

#### Exposed Ports

Your deployment must expose the following ports:

<table><thead><tr><th width="106.734375">Port</th><th>Description</th></tr></thead><tbody><tr><td><strong>22</strong></td><td>SSH service port — required for remote login.</td></tr><tr><td><strong>8888</strong></td><td>JupyterLab web interface port — required for browser access.</td></tr></tbody></table>

{% hint style="info" %}
These ports must be exposed; Without them, SSH and JupyterLab access will not be accessible.
{% endhint %}

#### Building a Custom Image

You can extend Yotta’s base images to install additional dependencies, frameworks, or internal tools.

#### Example Dockerfile

```dockerfile
FROM yottalabsai/pytorch:2.8.0-py3.11-cuda12.8.1-cudnn-devel-ubuntu22.04-2025081902

# Add custom dependencies or your own code
RUN pip install -U pip && \
    pip install pandas matplotlib
```

{% hint style="info" %}
💡 **Tip:** Extend the image to install additional ML libraries, enterprise tools, private model weights, or custom startup logic—without modifying the base environment.
{% endhint %}

#### Important Notes

⚠️ **Do Not Modify `/start.sh`**\
This script handles initialization for all services (SSH, Jupyter, etc.). Modifying or overriding it will cause the container to fail to start.&#x20;

⚠️ **Set `JUPYTER_PASSWORD`**\
Required for secure and successful JupyterLab initialization.

⚠️ **Expose Ports 22 and 8888**\
Mandatory for remote access and Jupyter web UI.

### Option 2: From Your Own Images

If you prefer to use your own base image, you must ensure that **SSH is properly installed and enabled**.\
The following example extends the latest vLLM image—simply replace the first line with your own base image.

```dockerfile
# Replace this base image with your own image 
FROM vllm/vllm-openai:latest

# Set environment variables to avoid interactive prompts during installation 
ENV DEBIAN_FRONTEND=noninteractive 

# Install OpenSSH server 

RUN apt-get update \
    && apt-get install -y openssh-server \
    && apt-get clean \
    && mkdir -p /run/sshd \
    && chmod 755 /run/sshd 
```

### Using a Custom Image in the Yotta Console

After building and pushing your custom image to your registry:

1. Log in to the **Yotta Console**.
2. Navigate to **Compute → Elastic Deployment** or **Pod Management**.
3. Select **Deploy**, then choose **Custom Image**.
4. Enter the image name and configuration.
5. Launch the Pod.

{% hint style="warning" %}
**Important:** If your base image does not automatically start `sshd`, add the following to your **Initialization Command**

`/usr/sbin/sshd -D &`

Then append any other commands you need.
{% endhint %}

<figure><img src="../.gitbook/assets/image (95).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure>

Your Pod will now run using your custom-built image — with full SSH and JupyterLab access.

<figure><img src="../.gitbook/assets/image (96).png" alt=""><figcaption></figcaption></figure>

### Summary

Yotta’s **Official Base Images** provide a optimized, GPU-ready foundation for AI workloads. By extending these images, developers can easily extend these images or bring their own, as long as SSH is enabled. Following the guidelines above ensures full compatibility with Yotta **Pods** and **Elastic Deployment**, enabling smooth development workflows with JupyterLab, SSH, and GPU acceleration.
