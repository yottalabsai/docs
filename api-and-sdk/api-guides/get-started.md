---
description: Start here with examples through api workflow listed below!
---

# Get started

#### 1. Create Credential for Private Image

```bash

curl -X POST https://api.yottalabs.ai/v2/container-registry-auths \
  -H "X-Api-Key: {X-Api-Key}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-private-registry",
    "type": "DOCKER_HUB",
    "username": "myuser",
    "password": "mypassword"
  }'
```

#### 2. Create VM

```bash
curl -X POST https://api.yottalabs.ai/v2/vms \
  -H "X-Api-Key: {X-Api-Key}" \
  -H "Content-Type: application/json" \
  -d '{
    "vmTypeId": 1,
    "region": "us-east-1",
    "name": "dev-vm",
    "isSpot": 0,
    "volumeMountPaths": {
      "123": "/mnt/data",
      "456": "/mnt/models"
    }
  }'
```

#### 3. Create Pod with GPU

<pre class="language-bash"><code class="lang-bash"><strong>curl -X POST https://api.yottalabs.ai/v2/pods \
</strong>  -H "X-Api-Key: {X-Api-Key}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "training-pod",
    "image": "pytorch/pytorch:2.0.0-cuda11.7-cudnn8-runtime",
    "gpuType": "NVIDIA_RTX_4090_24G",
    "gpuCount": 2,
    "containerVolumeInGb": 100
  }'
</code></pre>

#### 4. Create Elastic Endpoint (QUEUE Mode)

```bash
curl -X POST https://api.yottalabs.ai/v2/serverless \
  -H "X-Api-Key: {X-Api-Key}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "api-endpoint",
    "image": "vllm/vllm-openai:latest",
    "containerRegistryAuthId": 123,
    "resources": [
      {
        "region": "us-east-1",
        "gpuType": "NVIDIA_RTX_4090_24G",
        "gpuCount": 1
      }
    ],
    "workers": 2,
    "containerVolumeInGb": 120,
    "expose": {
      "port": 8000,
      "protocol": "http"
    },
    "serviceMode": "QUEUE"
  }'
```

#### 5. List and Monitor Resources

```wasm
##List VMs
curl https://api.yottalabs.ai/v2/vms?page=1&size=10 \
  -H "X-Api-Key: {X-Api-Key}"

# List Pods
curl https://api.yottalabs.ai/v2/pods \
  -H "X-Api-Key: {X-Api-Key}"

# List Endpoints
curl https://api.yottalabs.ai/v2/serverless \
  -H "X-Api-Key: {X-Api-Key}"
```
