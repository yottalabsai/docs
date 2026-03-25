---
icon: bolt
---

# Set up MCP server and Agent Skills

> **Objective:** Create a GPU Pod on YottaLabs using RTX 5090, PyTorch, and JupyterLab through natural language with Cursor's MCP integration.

***

### Quick Start Guide

Follow these 5 steps to create and access your GPU Pod:

<details>

<summary>Step 1: Prerequisites Check</summary>

Before you begin, ensure you have the following:

&#x20;**1.Node.js >= 18**

* **Check your version:** Run `node --version` in terminal
* **Not installed?** Download from [nodejs.org](https://nodejs.org/)

**2.Yotta API Key**

* Navigate to [Yotta Console](https://console.yottalabs.ai)
* Go to **Settings** → **Access Keys**
* Copy our apu keys securely

<figure><img src="../.gitbook/assets/image (170).png" alt=""><figcaption></figcaption></figure>

**3.Cursor IDE**

* Download and install the latest version of [Cursor](https://cursor.sh/)

</details>

<details>

<summary>Step 2: Configure MCP Server </summary>

Configure the Yotta MCP server in Cursor to enable natural language GPU management.

**Locate Configuration File**

Find or create the configuration file based on your operating system:

| Operating System | Configuration File Path                    |
| ---------------- | ------------------------------------------ |
| **Windows**      | `C:\Users\<YourUsername>\.cursor\mcp.json` |
| **macOS**        | `~/.cursor/mcp.json`                       |
| **Linux**        | `~/.cursor/mcp.json`                       |

**Add Configuration**

Create or edit the `mcp.json` file with the following content:

```json
{
  "mcpServers": {
    "yotta": {
      "command": "npx",
      "args": ["-y", "@yottascale/agent-native-infra"],
      "env": {
        "YOTTA_API_KEY": "your-yotta-api-key-here"
      }
    }
  }
}
```

{% hint style="info" %}
**Important:** Replace `your-yotta-api-key-here` with your actual Yotta API Key from Step 1.
{% endhint %}

**Restart Cursor**

After saving the configuration:

1. **Completely close** Cursor (all windows)
2. **Restart** Cursor
3. The Yotta MCP server will now be available

</details>

<details>

<summary>Step 3: Create Your GPU Pod </summary>

Now you can create a GPU Pod using natural language through Cursor Composer.

**Method: Natural Language (Recommended)**

1. **Open Cursor Composer** by pressing:
   * Windows/Linux: `Ctrl+I`
   * macOS: `Cmd+I`
2. **Type your request** in natural language:

```
Create a Yotta Pod named pytorch-rtx5090-jupyter using RTX 5090 GPU, 
PyTorch image, enable JupyterLab, and set password to yotta2025
```

3. **Let AI handle it** - The AI will call the `pod_create` tool with appropriate parameters

**Key Parameters Reference ( just an example )**&#x20;

<table><thead><tr><th width="155.24237060546875">Parameter</th><th width="112.6060791015625">Description</th><th>Example Value</th></tr></thead><tbody><tr><td><code>name</code></td><td>Pod name</td><td><code>pytorch-rtx5090-jupyter</code></td></tr><tr><td><code>image</code></td><td>Docker image</td><td><code>yottalabsai/pytorch:2.8.0-py3.11-cuda12.8.1-cudnn-devel-ubuntu22.04-2025081902</code></td></tr><tr><td><code>gpuType</code></td><td>GPU type</td><td><code>NVIDIA_RTX_5090_32G</code></td></tr><tr><td><code>gpuCount</code></td><td>Number of GPUs</td><td><code>1</code></td></tr><tr><td><code>environmentVars</code></td><td>Environment variables</td><td><code>JUPYTER_PASSWORD</code> (required)</td></tr><tr><td><code>expose</code></td><td>Ports to expose</td><td><code>22</code> (SSH), <code>8888</code> (JupyterLab)</td></tr><tr><td><code>regions</code></td><td>Preferred regions</td><td><code>["us-east-1", "us-east-2"]</code></td></tr></tbody></table>

</details>

<details>

<summary>Step 4: Wait for Pod Creation </summary>

After submitting your request, you'll receive a response confirming the pod creation.

**✅ Success Response Example**

```json
{
  "id": "420522713875330018",
  "name": "pytorch-rtx5090-jupyter",
  "gpuType": "NVIDIA_RTX_5090_32G",
  "gpuDisplayName": "RTX 5090",
  "gpuCount": 1,
  "singleCardVramInGb": 32,
  "singleCardPrice": "0.65",
  "status": "INITIALIZE",
  "environmentVars": [
    {"key": "JUPYTER_PASSWORD", "value": "yotta2025"}
  ],
  "expose": [
    {"port": 22, "protocol": "SSH"},
    {"port": 8888, "protocol": "HTTP"}
  ]
}
```

&#x20;**Status Progression**

Your pod will go through these states:

1. **INITIALIZE** → Pod is being created
2. **RUNNING** → Pod is ready to use (usually takes 1-2 minutes)

</details>

<details>

<summary>Step 5: Access Your Pod</summary>

Once the pod status changes to **RUNNING**, you can access it.

&#x20;**Via Yotta Console**

1. Open [Yotta Console](https://console.yottalabs.ai)
2. Navigate to **Compute** → **Pods**
3. Click on your pod: `pytorch-rtx5090-jupyter`
4. View the access URLs in the details panel

> Get the SSH address from the pod details in the console.

</details>

***

### Pod Management Commands

Once your pod is running, you can manage it using **natural language** commands in Cursor Composer:

#### &#x20;List All Pods

```
List all my Yotta pods
```

#### Get Pod Details

```
Get details for pod pytorch-rtx5090-jupyter
```

#### Pause Pod (Stop Billing)

```
Pause pod pytorch-rtx5090-jupyter
```

{% hint style="info" %}
**Tip:** Pause pods when not in use to save costs!
{% endhint %}

#### &#x20;Delete Pod

```
Delete pod pytorch-rtx5090-jupyter
```

{% hint style="info" %}
**Warning:** This action is irreversible!
{% endhint %}

