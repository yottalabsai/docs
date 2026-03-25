# Compute

<details>

<summary>What is a Pod?</summary>

A **Pod** is a container-based compute environment with GPUs on Yotta Labs, designed for running AI workloads quickly and reproducibly.

Pods run on **containers**, which means you can choose from **official, ready-to-use images** provided by Yotta Labs (pre-configured for common AI workflows), or you can **build and use your own custom Docker image** for full control over your runtime, dependencies, and environment.

</details>

<details>

<summary>What is a Virtual Machine?</summary>

A **Virtual Machine (VM)** on Yotta Labs is a **GPU-powered VM instance** that gives you a more traditional server environment.

It comes with a **basic operating system** and **CUDA pre-installed**, and you’re free to install and configure everything else yourself—such as frameworks (PyTorch/TensorFlow), drivers, libraries, and your own software stack.

</details>

<details>

<summary>What is the difference between Pods and VMs?</summary>

**Pods** are best when you want a **container-native workflow** with quick setup and portable environments.

**VMs** are best when you want **full OS-level control** and prefer managing the software stack yourself.

In general:

* Choose **Pods** for faster iteration and containerized workflows
* Choose **VMs** for maximum flexibility and full system customization

</details>

<details>

<summary>How do I create a Pod or VM?</summary>

You can create a Pod or VM directly in the **Yotta Console**.

1. Click **Launch Console**
2. Go to **Compute**
3. Choose **Pods** or **Virtual Machines**
4. Select your desired configuration (GPU / CPU / memory / region)
5. Click **Create** to launch your instance

</details>

<details>

<summary>How can I access the Pod/VM I created?</summary>

We offer two access methods:

* **Jupyter Notebook**: Go to the instance page and click **IDE** under the **Connect** column to open the built-in Jupyter environment.
* **SSH**: Use the **private key associated with your Yotta account** to connect to your Pod or VM via **SSH**.

</details>

<details>

<summary>How can I access my private key?</summary>

You can find your public and private keys under **Settings → Access Keys** in the Yotta Console.

Download your **private key** to your local machine, then use it to **SSH into your on-demand Pod or VM**.

</details>

<details>

<summary>Can I use my own container image for Pods?</summary>

Yes. Pods support both:

* **Official images** (ready to use)
* **Custom images** you build and maintain yourself

This makes it easy to standardize environments across teams and keep workloads reproducible.

</details>

<details>

<summary>What is included by default on a VM?</summary>

Yotta VMs come with:

* A **basic operating system**
* **CUDA pre-installed**

You are responsible for installing any additional software you need, such as:

* PyTorch / TensorFlow
* vLLM / SGLang
* System libraries and dependencies
* Your own application code and services

</details>

<details>

<summary>Is data persistent on Pods and VMs?</summary>

Persistence depends on the storage option you choose.

In general:

* If you use **ephemeral storage**, data may not persist after the instance is terminated.
* For long-running projects and reusable datasets/models, we recommend using **persistent storage**.

</details>

<details>

<summary>How is Pod/VM usage billed?</summary>

Yotta Labs uses **usage-based pricing**. You only pay for the compute resources you actually consume.

Compute usage is metered with **per-second precision**, and there are **no long-term commitments** required.

</details>
