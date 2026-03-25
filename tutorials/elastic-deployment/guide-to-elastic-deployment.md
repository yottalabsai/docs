---
icon: location-arrow
---

# Guide to Elastic Deployment

### **Introduction**

This comprehensive guide explains how to use YottaLabs’ **Elastic Deployment** feature to quickly and reliably deploy **Qwen3-0.6B** model as containerized services  in a production environment.

***

### **1. Background**

Yotta Labs is a one-stop platform for AI application development. This Elastic Deployment feature is specifically designed for low-latency, high-concurrency inference services, supporting:

* **Auto-scaling**
* **Self-healing (Fault tolerance)**
* **Multi-GPU scheduling**

Unlike traditional static deployments, Elastic Deployment abstracts computing units (**Workers**) into stateless service instances that can be dynamically scaled.&#x20;

***

### **2. Core Deployment Steps**

#### **Step 1: Preparation – API Key & Console Access**

![img](https://vs-oss.cqywc.com/prod/videoseek/snapshot/856271224265768960/856274557722427392_00002.jpg)

Before deploying, you must obtain your identity credentials:

1. Log in to the **YottaLabs Console**.
2. Navigate to **Settings → Access Keys**.
3. Generate and copy your **API Key**. This key is required for all automated operations and management interfaces.
4. Return to the main menu and click on **Elastic Deployment** to begin.

#### **Step 2: Image Configuration – Public vs. Private Registries**

![img](https://vs-oss.cqywc.com/prod/videoseek/snapshot/856271224265768960/856274557722427392_00003.jpg)

The image source is the foundation of your deployment. Yotta Labs supports two categories:

* **Docker Hub (Public):** Simply provide the full image name, e.g., `myorg/qwen-vllm:2.3.1-cu121`.
* **Other (Private):** For private registries , you must provide the registry URL and valid **Credentials**.

#### **Step 3: Service Modes & GPU Allocation**

Choose modes that matches your business logic:

<table data-header-hidden><thead><tr><th width="161"></th><th width="185.6666259765625"></th><th></th></tr></thead><tbody><tr><td><strong>SERVICE MODE</strong></td><td><strong>BEST USE CASE</strong></td><td><strong>LOGIC</strong></td></tr><tr><td><strong>ALB</strong></td><td>Web APIs / Chatbots</td><td>Automatically distributes traffic.</td></tr><tr><td><strong>Queue</strong></td><td>Batch Processing</td><td>Workers pull tasks from a queue; ideal for non-real-time tasks.</td></tr><tr><td><strong>Custom</strong></td><td>Advanced Integration</td><td>Opens raw ports for users with their own load balancers.</td></tr></tbody></table>

**For more information here, see our** [**official docs of service mode**](../../products/serverless/service-mode.md)

**GPU Selection:**

<figure><img src="../../.gitbook/assets/image (30).png" alt="" width="563"><figcaption></figcaption></figure>

You can specify the **GPU Type** (e.g. RTX 4090), **GPU Count** per worker (1–3), and **VRAM** requirements.

#### **Step 4: Elastic Mechanism – Worker Lifecycle**

<figure><img src="../../.gitbook/assets/image (31).png" alt="" width="563"><figcaption></figcaption></figure>

* **Declarative Scaling:** If you set the target to 2 Workers, YottaLabs ensures 2 Workers are always running.
* **Automatic Recovery:** If a Worker crashes/is terminated accidentally, the system will automatically spins up a new instance within seconds.

***

### **3. Deployment and validation**

Once the status changes to **Running**, YottaLabs generates a unique HTTPS URL. Copy the url provided in the box above.

<figure><img src="../../.gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

Use a `curl` command to test the service, for example:

```
curl -X POST "https://32tkdcwyscmx.yottadeos.com/v1/chat/completions" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer <YOUR_API_KEY>" \
-d '{
  "model": "Qwen/Qwen3-0.6B",
  "messages": [
    {"role": "system", "content": "You are a helpful AI assistant."},
    {"role": "user", "content": "Explain AI in simple terms."}
  ],
  "temperature": 0.7,
  "max_tokens": 512
}'
```

A successful response confirms that the VLLM inference engine, Tokenizer, and CUDA acceleration path are all functioning correctly.

<figure><img src="../../.gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>

