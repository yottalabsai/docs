---
description: >-
  Pods are the core compute units on Yotta Platform. They allow users to deploy,
  manage, and connect to isolated GPU workloads across various hardware through
  the Console or the API interface.
icon: bolt
---

# GPU Pods

## 💻 Managing Pods via Console

#### Pod Page Overview

When entering the **Pods** page:

*   The system by default displays Pods in **In Progress** state, including:

    * `Initializing` – Resources are being allocated; the Pod is deploying.If you&#x20;
    * `Running` – The Pod is running normally.
    * `Terminating` – The Pod is being terminated; resources are being reclaimed.

    <figure><img src="../.gitbook/assets/image (145).png" alt="" width="375"><figcaption></figcaption></figure>
*   There are buttons you can use at the bottom:

    *   &#x20;:link: `Connect` You can connect your machine to specific ports, such as 8888 for Jupyter Notebook. We currently support both **SSH** and **HTTP** ports.

        <figure><img src="../.gitbook/assets/image (148).png" alt="" width="375"><figcaption></figcaption></figure>

    <figure><img src="../.gitbook/assets/image (149).png" alt="" width="375"><figcaption></figcaption></figure>

    * :notepad\_spiral: `Log`  You can check the container logs to view its current status and identify any errors or issues.



    <figure><img src="../.gitbook/assets/image (153).png" alt="" width="375"><figcaption></figcaption></figure>

    * :chart\_with\_upwards\_trend:`Metrics`  This provides real-time monitoring of GPU, CPU, memory, and storage usage to help you track system performance and resource utilization.

    <figure><img src="../.gitbook/assets/image (155).png" alt=""><figcaption></figcaption></figure>
*   Click the **History** tab to view Pods that have completed within the last 24 hours, which includes:

    * `Terminated` – The Pod has been deleted.
    * `Failed` – Deployment failed. Common causes:
      * Insufficient system resources
      * Invalid image configuration

    <figure><img src="../.gitbook/assets/image (147).png" alt="" width="270"><figcaption></figcaption></figure>
* There is a search bar where you can use Pod name to find your Pod (fuzzy search supported). You can also use **Pod Status** or **GPU Type** to filter Pods.&#x20;

## ⚙️ Deploying a Pod

#### Step-by-Step Guide

*   **Navigate to:** `Compute → Pods`

    <figure><img src="../.gitbook/assets/image (157).png" alt=""><figcaption></figcaption></figure>
* **Click Deploy** (top right).  You’ll enter the **GPU Selection** page.

<figure><img src="../.gitbook/assets/image (158).png" alt=""><figcaption></figcaption></figure>

*   **Select GPU Type**

    * Choose a GPU model suitable for your workload.


*   **Configure Pod**

    * Fill in required parameters (fields marked with **`*`** are mandatory).

    <figure><img src="../.gitbook/assets/image (160).png" alt=""><figcaption></figcaption></figure>

**Image Requirements**

Click **Edit** next to image name to further configure your image. We provided a list of official images compiled by Yotta Labs. Also, we allow users to select custom images including both **Public Images** and **Private Images**.

Here are are few requirements if you want to build your custom image:

* Must be compiled for **x86** architecture&#x20;
* Must be **Debian/Ubuntu**

5. **Deploy**\
   Click **Deploy** to complete the process.

<figure><img src="../.gitbook/assets/image (161).png" alt="" width="256"><figcaption></figcaption></figure>

## 💾 **System Volume**

The System Volume would automatically mount a list of system directories on the created Pod. This ensures that software, configurations, and data stored within these directories are persistent even if the Pod is edited or restarted.&#x20;

#### **Supported Directories**&#x20;

Read and write operations to the following directories will be persistent:

<table data-header-hidden data-full-width="false"><thead><tr><th width="169.5" align="center" valign="middle"></th><th align="center"></th></tr></thead><tbody><tr><td align="center" valign="middle"><strong>Directory</strong></td><td align="center"><strong>Brief Description</strong></td></tr><tr><td align="center" valign="middle"><code>/home</code></td><td align="center">User home directories; user-level configs and data.</td></tr><tr><td align="center" valign="middle"><code>/root</code></td><td align="center">Root user home directory; scripts and temp data.</td></tr><tr><td align="center" valign="middle"><code>/var</code></td><td align="center">Variable files (logs, caches, runtime data).</td></tr><tr><td align="center" valign="middle"><code>/run</code></td><td align="center">Runtime status files (PIDs, sockets).</td></tr><tr><td align="center" valign="middle"><code>/etc</code></td><td align="center">System and service configuration files.</td></tr><tr><td align="center" valign="middle"><code>/usr</code></td><td align="center">System-level apps, libraries, and runtime components.</td></tr></tbody></table>

#### **Size Requirements**

To ensure that the Pod can launch and run smoothly, we recommend using the following rule to decide the size of your system volume:

> The size of the system volume needs to ≥ Image Size × 3

**Example:**

* **Image:** PyTorch base image (10 GiB)
* **Recommended System Volume Size:** At least 30 GiB

{% hint style="info" %}
The "**For Development**" button is automatically turned on when you are creating a pod.

You can find it and change the settings  beside the pod name bar.

System volume is set to 100GB by default.
{% endhint %}

#### **Recommended Use Cases:**

* **Preserving Environments:** Retaining toolchains or dependencies (e.g., pip packages) after a Pod rebuild.
* **Persisting Configurations:** Saving changes made in `/etc`.
* **Retaining Logs:** Keeping logs in `/var` for a specific period.

<figure><img src="../.gitbook/assets/image (162).png" alt="" width="375"><figcaption></figcaption></figure>

## 🔌 Connecting to Your Pod

Once the Pod is launched:

* Click the **Connect** button on the Pod card to view exposed services.
* Availability depends on the **port configuration** defined at deployment.
* When the container port is **Ready**, the status will update automatically.

<figure><img src="../.gitbook/assets/image (165).png" alt="" width="375"><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (163).png" alt="" width="375"><figcaption></figcaption></figure>

## 📜 Viewing Logs

* Click **Logs** on the Pod card to view both:
  * **System Logs** (platform-level)
  * **Container Logs** (application-level)

This helps with debugging deployment or runtime issues.

<figure><img src="../.gitbook/assets/image (166).png" alt=""><figcaption></figcaption></figure>

## 🧊 Pausing or Terminating Pods

#### 🔸 Pause

If you only need to suspend temporarily:

* Click **Pause** on the Pod card.
* Only **Volume** storage will continue to incur charges.
* You can **Run** to restart anytime.
* Pods can be **edited** while paused.

#### 🔸 Terminate

If you want to remove the Pod completely:

* Click the **“...”** on the Pod card → choose **Terminate**.
* The Pod will be **permanently deleted** and **no longer billed**.
* Terminated Pods **cannot be edited or restarted**.

## ✏️ Editing a Pod

1. Go to **Compute → Pods** and locate the Pod.
2. Click **Pause** and wait until the Pod enters **Stopped** state.
3. Click **“...” → Edit**, modify configurations, and save.
4. Click **Run** to restart the Pod with the new settings.

## 📈 Pod Status Reference

| Status          | Description                                                |
| --------------- | ---------------------------------------------------------- |
| **Initialize**  | Resource allocation in progress; Pod deploying             |
| **Running**     | Pod is running                                             |
| **Stopping**    | Pausing in progress; resources reclaiming                  |
| **Stopped**     | Pod is paused                                              |
| **Terminating** | Termination in progress; resources reclaiming              |
| **Terminated**  | Pod fully terminated                                       |
| **Failed**      | Deployment failed (insufficient resources / invalid image) |

## 💰 Pricing & Billing

#### Formula

```
Pod hourly cost = (GPU unit price × number of GPUs)
                + (Disk hourly rate × GB size)
                + (Volume hourly rate × GB size)
```

#### Deduction Rules

* Billing starts once the Pod is **Running**.
* When balance nears **$0**, all active Pods will be **terminated automatically**.
* To avoid charges:
  * Use **Pause** to temporarily suspend (still charges for persistent volumes).
  * Use **Terminate** to completely stop billing.

| Action        | Billing Behavior                             |
| ------------- | -------------------------------------------- |
| **Pause**     | Charges continue for Volumes (Stopped state) |
| **Terminate** | No charges (Terminated state)                |

***

## 🧩 Managing Pods via OpenAPI

You can also manage Pods programmatically via Yotta Labs’ **OpenAPI**.

#### API Reference

* [📘 API & SDK Documentation](https://docs.yottalabs.ai/yotta-labs/api-and-sdk/api-guides)

> **Tip:**\
> Always review the API documentation before calling endpoints to avoid common request errors (invalid parameters, insufficient balance, etc.).

## 🧱 Example Use Cases

* **Automated Pod Deployment** via Python SDK
* **Monitoring Pod Logs** using API polling
* **Scaling Workloads** across multiple GPU types
* **Integrating with CI/CD** to trigger training jobs automatically

***

## 🪄 Best Practices

* Use **Pause** instead of **Terminate** for short-term downtime.
* Monitor balance regularly to prevent auto-termination.
* Always verify image compatibility (x86 / Ubuntu-based).
* For debugging, prefer checking container logs first.

***

## 🧩 Related Docs

* [Pod API Reference](https://docs.yottalabs.ai/yotta-labs/api-and-sdk/api-guides)
* [Billing](https://docs.yottalabs.ai/yotta-labs/products/billing)

