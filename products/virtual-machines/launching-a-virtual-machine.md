# Launching a Virtual Machine

### Overview&#x20;

YottaLabs provides an intuitive interface for launching virtual machines with powerful GPU resources. Whether you're training machine learning models, running scientific simulations, or performing data analysis, this platform makes it easy to configure the exact resources you need.

***

<figure><img src="../../.gitbook/assets/image (132).png" alt=""><figcaption></figcaption></figure>

#### 1. Region Selector

<figure><img src="../../.gitbook/assets/image (133).png" alt="" width="117"><figcaption></figcaption></figure>

**What it does:** Selects the geographic location where your virtual machine will be hosted.

**Why it matters:**

* **Latency**: Choose a region close to you for faster response times
* **Availability**: Different regions may have different GPU availability

***

#### 2. Network Storage

<figure><img src="../../.gitbook/assets/image (134).png" alt="" width="375"><figcaption></figcaption></figure>

**What it does:** Configures persistent network-attached storage for your virtual machine.

**Storage types typically include:**

* **Standard SSD**: Good for general purpose workloads
* **High-performance SSD**: For I/O intensive applications
* **Archive storage**: For infrequently accessed data

**Why it matters:**&#x59;our instance's local storage is ephemeral (lost when instance stops)!

***

#### 3. VRAM Slider

**What it does:** Filters instance types based on GPU memory (VRAM) requirements.

**Why it matters:**

* Different models require different amounts of GPU memory
* Training large language models needs 40GB+ VRAM
* Small computer vision models might only need 8-16GB

**Example use cases:**

* **1-8 GB**: Small datasets, inference, light training
* **16-24 GB**: Medium neural networks, most computer vision tasks
* **40-80 GB**: Large language models, high-resolution image processing
* **80+ GB**: Multi-GPU training, extremely large models

***

#### 4. RAM Slider

**What it does:** Filters instance types based on system RAM .

{% hint style="info" %}
**VRAM vs RAM**

RAM is the CPU's general-purpose workspace for running the OS and apps, while VRAM is the GPU's high-speed dedicated memory used exclusively for rendering textures, 3D graphics, and video frames. Essentially, RAM manages the "logic" of your computer, whereas VRAM handles the "visuals."

&#x20;**For deep learning, a good rule of thumb is 2-4x your VRAM in system RAM.**
{% endhint %}

**Why it matters:**

* Data preprocessing often happens in system RAM
* Large datasets need to fit in memory
* Some applications are RAM-intensive even without GPUs

***

#### Provider Selection

Choose different GPU providers:

<figure><img src="../../.gitbook/assets/image (167).png" alt=""><figcaption></figcaption></figure>

***

### GPU Configuration

#### Multi-GPU Selection Buttons

* Not all code can utilize multiple GPUs without modification
* You'll need to implement data parallelism or model parallelism
* PyTorch: Use `DataParallel` or `DistributedDataParallel`
* TensorFlow: Use distribution strategies
* Monitor GPU utilization - don't pay for GPUs you're not using!

***

#### Operating System Selection

**What it does:** Selects the operating system for your virtual machine.

**Ubuntu 22.04 LTS**: Long-term support, most stabled configurations

<figure><img src="../../.gitbook/assets/image (137).png" alt=""><figcaption></figcaption></figure>

**Ubuntu 22.04 LTS** (shown above):

* **LTS** = Long Term Support (5 years of updates)
* Well-supported by AI/ML tools
* Most documentation assumes Ubuntu
* Native NVIDIA CUDA support

***

#### Purchase Option

<figure><img src="../../.gitbook/assets/image (138).png" alt="" width="563"><figcaption></figcaption></figure>

**On-Demand** instances are like traditional cloud computing - you get exactly what you pay for, when you need it, for as long as you need it.

**Spot instances** are spare capacity that cloud providers sell at a discount. The catch? They can be interrupted against your will.
