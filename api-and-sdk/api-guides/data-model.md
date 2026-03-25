# Data Model

### Base Response Models

#### StandardResponse

All API responses follow this format:

```json
{
  "message": "string",
  "code": integer,
  "data": object
}
```

| Field   | Type    | Description                                            |
| ------- | ------- | ------------------------------------------------------ |
| message | string  | Response message                                       |
| code    | integer | Response code (see Response Code table)                |
| data    | object  | Response data (specific structure depends on endpoint) |

#### ResponseCode

| Code  | Description                |
| ----- | -------------------------- |
| 10000 | Success                    |
| 10001 | Invalid request parameters |
| 10002 | System error               |
| 10007 | Record not exist           |
| 11000 | VM not found               |
| 11001 | VM status invalid          |
| 11002 | No terminate permissions   |
| 12001 | Resource status invalid    |
| 13000 | Pod not found              |
| 14000 | Endpoint not found         |
| 24000 | Serverless unavailable     |
| 24001 | Serverless does not exist  |
| 24011 | Task does not exist        |

***

### Container Registry Credential Models

#### Credential

Container registry credential object

```json
{
  "id": 123,
  "name": "my-docker-registry",
  "type": "DOCKER_HUB",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

| Field     | Type    | Required | Description                                                 |
| --------- | ------- | -------- | ----------------------------------------------------------- |
| id        | integer | Yes      | Credential ID                                               |
| name      | string  | Yes      | Credential name                                             |
| type      | string  | Yes      | Registry type: `DOCKER_HUB`, `GCR`, `ECR`, `ACR`, `PRIVATE` |
| createdAt | string  | Yes      | Creation time (ISO 8601)                                    |

#### CreateCredentialRequest

```json
{
  "name": "my-docker-registry",
  "type": "DOCKER_HUB",
  "username": "myuser",
  "password": "mypassword"
}
```

| Field    | Type   | Required | Description             |
| -------- | ------ | -------- | ----------------------- |
| name     | string | Yes      | Credential name         |
| type     | string | Yes      | Registry type           |
| username | string | Yes      | Registry username       |
| password | string | Yes      | Registry password/token |

#### UpdateCredentialRequest

```json
{
  "name": "updated-name",
  "username": "newuser",
  "password": "newpassword"
}
```

| Field    | Type   | Required | Description         |
| -------- | ------ | -------- | ------------------- |
| name     | string | No       | New credential name |
| username | string | No       | New username        |
| password | string | No       | New password/token  |

***

### Virtual Machine Models

#### VM

```json
{
  "id": 456,
  "name": "my-vm",
  "status": "running",
  "gpuDisplayName": "NVIDIA RTX A4000",
  "ipAddress": "192.168.1.100",
  "cpuCores": 8,
  "memoryInGb": "32",
  "gpuCount": 1,
  "gpuMemoryInGb": "16",
  "region": "us-east-1",
  "storageInGb": "100",
  "sshTemplate": "ssh root@192.168.1.100",
  "createdAt": "2024-01-15T10:30:00Z",
  "isSpot": 0
}
```

| Field          | Type    | Required | Description                                                   |
| -------------- | ------- | -------- | ------------------------------------------------------------- |
| id             | integer | Yes      | VM ID                                                         |
| name           | string  | Yes      | VM name                                                       |
| status         | string  | Yes      | VM status: `running`, `initializing`, `stopped`, `terminated` |
| gpuDisplayName | string  | Yes      | GPU display name                                              |
| ipAddress      | string  | Yes      | IP address                                                    |
| cpuCores       | integer | Yes      | Number of CPU cores                                           |
| memoryInGb     | string  | Yes      | Memory size (GB)                                              |
| gpuCount       | integer | Yes      | Number of GPUs                                                |
| gpuMemoryInGb  | string  | Yes      | GPU memory size (GB)                                          |
| region         | string  | Yes      | Region code                                                   |
| storageInGb    | string  | Yes      | Storage size (GB)                                             |
| sshTemplate    | string  | Yes      | SSH connection template                                       |
| createdAt      | string  | Yes      | Creation time                                                 |
| isSpot         | integer | Yes      | Is Spot instance: 0=On-Demand, 1=Spot                         |

#### CreateVMRequest

```json
{
  "vmTypeId": 1,
  "region": "us-east-1",
  "name": "my-vm",
  "isSpot": 0,
  "volumeMountPaths": {
    "123": "/mnt/data",
    "456": "/mnt/models"
  }
}
```

| Field            | Type    | Required | Description                      |
| ---------------- | ------- | -------- | -------------------------------- |
| vmTypeId         | integer | Yes      | VM type ID                       |
| region           | string  | Yes      | Region code (e.g., "us-east-1")  |
| name             | string  | Yes      | VM display name                  |
| isSpot           | integer | No       | 0=On-Demand, 1=Spot (default: 0) |
| volumeMountPaths | map     | No       | Volume ID to mount path mapping  |

#### UpdateVMRequest

```json
{
  "name": "updated-vm-name"
}
```

| Field | Type   | Required | Description |
| ----- | ------ | -------- | ----------- |
| name  | string | Yes      | New VM name |

#### VMType

```json
{
  "gpuType": "NVIDIA_RTX_4090_24G",
  "regions": [
    {
      "region": "us-east-1",
      "regionName": "US East",
      "available": true,
      "vmTypeId": 1
    }
  ]
}
```

| Field                 | Type    | Required | Description                                 |
| --------------------- | ------- | -------- | ------------------------------------------- |
| gpuType               | string  | Yes      | GPU type code                               |
| regions               | array   | Yes      | List of available regions for this GPU type |
| regions\[].region     | string  | Yes      | Region code                                 |
| regions\[].regionName | string  | Yes      | Region name                                 |
| regions\[].available  | boolean | Yes      | Is available                                |
| regions\[].vmTypeId   | integer | Yes      | VM type ID                                  |

***

### Pod Models

#### Pod

```json
{
  "id": 789,
  "name": "my-pod",
  "image": "pytorch/pytorch:2.0.0-cuda11.7-cudnn8-runtime",
  "imageRegistry": "https://index.docker.io/v1/",
  "gpuType": "NVIDIA_RTX_4090_24G",
  "gpuDisplayName": "RTX 4090",
  "gpuCount": 1,
  "status": "RUNNING",
  "createdAt": 1705306200000,
  "sshCmd": "ssh root@pod-ip -p 10022"
}
```

| Field          | Type    | Required | Description              |
| -------------- | ------- | -------- | ------------------------ |
| id             | integer | Yes      | Pod ID                   |
| name           | string  | Yes      | Pod name                 |
| image          | string  | Yes      | Docker image             |
| imageRegistry  | string  | Yes      | Image registry URL       |
| gpuType        | string  | Yes      | GPU type                 |
| gpuDisplayName | string  | Yes      | GPU display name         |
| gpuCount       | integer | Yes      | Number of GPUs           |
| status         | string  | Yes      | Pod status               |
| createdAt      | integer | Yes      | Creation time (epoch ms) |
| sshCmd         | string  | Yes      | SSH connection command   |

#### CreatePodRequest

```json
{
  "name": "my-pod",
  "regions": ["us-east-1", "us-west-1"],
  "image": "pytorch/pytorch:2.0.0-cuda11.7-cudnn8-runtime",
  "containerRegistryAuthId": 123,
  "gpuType": "NVIDIA_RTX_4090_24G",
  "gpuCount": 1,
  "minSingleCardVramInGb": 24,
  "minSingleCardRamInGb": 32,
  "minSingleCardVcpu": 8,
  "containerVolumeInGb": 50,
  "persistentVolumeInGb": 100,
  "persistentMountPath": "/mnt/data",
  "initializationCommand": "pip install -r requirements.txt",
  "environmentVars": [
    { "key": "API_KEY", "value": "secret" }
  ],
  "expose": [
    { "port": 8080, "protocol": "http" }
  ],
  "persistentVolumes": [
    {
      "volumeId": 123,
      "mountPath": "/mnt/volume",
      "needBackup": false
    }
  ]
}
```

| Field                   | Type    | Required | Description                                           |
| ----------------------- | ------- | -------- | ----------------------------------------------------- |
| name                    | string  | Yes      | Pod name (1-255 characters)                           |
| regions                 | array   | No       | Accepted region codes for scheduling                  |
| image                   | string  | Yes      | Docker image                                          |
| imageRegistry           | string  | No       | Docker registry URL (default: Docker Hub)             |
| containerRegistryAuthId | integer | No       | Container registry credential ID (for private images) |
| imagePublicType         | string  | No       | Image type: `PUBLIC` or `PRIVATE` (default: PUBLIC)   |
| resourceType            | string  | No       | Resource type: `GPU` or `CPU` (default: GPU)          |
| gpuType                 | string  | Yes      | GPU type (e.g., "NVIDIA\_RTX\_4090\_24G")             |
| gpuCount                | integer | Yes      | Number of GPUs (must be power of 2)                   |
| minSingleCardVramInGb   | integer | No       | Minimum single card VRAM (GB)                         |
| minSingleCardRamInGb    | integer | No       | Minimum single card RAM (GB)                          |
| minSingleCardVcpu       | integer | No       | Minimum single card vCPU count                        |
| shmInGb                 | integer | No       | Shared memory size (GB)                               |
| containerVolumeInGb     | integer | No       | Container volume size (GB)                            |
| persistentVolumeInGb    | integer | No       | Persistent volume size (GB)                           |
| persistentMountPath     | string  | No       | Persistent volume mount path                          |
| initializationCommand   | string  | No       | Initialization command                                |
| environmentVars         | array   | No       | Environment variables array                           |
| expose                  | array   | No       | Port exposure configuration                           |
| persistentVolumes       | array   | No       | Persistent volumes configuration                      |

***

### Elastic Endpoint Models

#### Endpoint

```json
{
  "id": "378888638324150969",
  "name": "llama-inference",
  "image": "vllm/vllm-openai:latest",
  "status": "RUNNING",
  "totalWorkers": 2,
  "runningWorkers": 2,
  "cost": 1.5,
  "serviceMode": "QUEUE",
  "webhook": null
}
```

| Field          | Type    | Required | Description                            |
| -------------- | ------- | -------- | -------------------------------------- |
| id             | string  | Yes      | Endpoint ID                            |
| name           | string  | Yes      | Endpoint name                          |
| image          | string  | Yes      | Docker image                           |
| status         | string  | Yes      | Endpoint status                        |
| totalWorkers   | integer | Yes      | Total number of workers                |
| runningWorkers | integer | Yes      | Number of running workers              |
| cost           | number  | Yes      | Cost                                   |
| serviceMode    | string  | Yes      | Service mode: `ALB`, `QUEUE`, `CUSTOM` |
| webhook        | string  | No       | Webhook URL                            |

#### CreateEndpointRequest

```json
{
  "name": "llama-inference",
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
  "environmentVars": [
    { "key": "MODEL", "value": "llama-3-8b" }
  ],
  "expose": {
    "port": 8000,
    "protocol": "http"
  },
  "serviceMode": "QUEUE",
  "webhook": "https://webhook.example.com/status"
}
```

| Field                   | Type    | Required | Description                                           |
| ----------------------- | ------- | -------- | ----------------------------------------------------- |
| name                    | string  | Yes      | Endpoint name (max 20 characters, start with letters) |
| image                   | string  | Yes      | Docker image                                          |
| imageRegistry           | string  | No       | Docker registry URL (default: Docker Hub)             |
| containerRegistryAuthId | integer | No       | Container registry credential ID                      |
| resources               | array   | Yes      | GPU resource configuration                            |
| workers                 | integer | Yes      | Number of workers                                     |
| containerVolumeInGb     | integer | Yes      | Container volume size (min 20 GB)                     |
| environmentVars         | array   | No       | Environment variables                                 |
| expose                  | object  | No       | Port exposure configuration                           |
| serviceMode             | string  | Yes      | Service mode                                          |
| webhook                 | string  | No       | Webhook URL (max 512 characters)                      |

#### UpdateEndpointRequest

```json
{
  "name": "updated-name",
  "resources": [
    {
      "region": "us-east-1",
      "gpuType": "NVIDIA_RTX_4090_24G",
      "gpuCount": 1
    }
  ],
  "workers": 4,
  "containerVolumeInGb": 120,
  "envVars": [
    { "key": "MODEL", "value": "llama-3-70b" }
  ]
}
```

| Field                 | Type    | Required | Description                       |
| --------------------- | ------- | -------- | --------------------------------- |
| name                  | string  | Yes      | Endpoint name                     |
| resources             | array   | Yes      | GPU resource configuration        |
| workers               | integer | Yes      | Number of workers (min 1)         |
| containerVolumeInGb   | integer | Yes      | Container volume size (min 20 GB) |
| minSingleCardVramInGb | integer | No       | Minimum GPU single card VRAM (GB) |
| minSingleCardVcpu     | integer | No       | Minimum GPU single card vCPU      |
| minSingleCardRamInGb  | integer | No       | Minimum GPU single card RAM (GB)  |
| credentialId          | integer | No       | Container registry credential ID  |
| initializationCommand | string  | No       | Initialization command            |
| environmentVars       | array   | No       | Environment variables             |
| expose                | object  | No       | Port exposure configuration       |
| webhook               | string  | No       | Webhook URL (max 512 characters)  |

#### Worker

```json
{
  "workerId": "worker-001",
  "status": "running",
  "createdAt": 1705306200000
}
```

| Field     | Type    | Required | Description   |
| --------- | ------- | -------- | ------------- |
| workerId  | string  | Yes      | Worker ID     |
| status    | string  | Yes      | Worker status |
| createdAt | integer | Yes      | Creation time |

***

### Task Models

#### Task

```json
{
  "taskId": "my_task_001",
  "endpointId": 456,
  "endpointName": "llama-inference",
  "status": "SUCCESS",
  "workerUrl": "https://worker.example.com",
  "webhook": "https://webhook.example.com/callback",
  "deliveryStatus": "SUCCESS",
  "deliveryAttempts": 1,
  "error": null,
  "input": { "prompt": "Hello, world!" },
  "output": { "response": "Hi there!" },
  "headers": { "Authorization": "Bearer token123" },
  "createdAt": 1705306200000,
  "updatedAt": 1705306260000,
  "deliveredAt": 1705306260000
}
```

| Field            | Type    | Required | Description                 |
| ---------------- | ------- | -------- | --------------------------- |
| taskId           | string  | Yes      | Task ID                     |
| endpointId       | integer | Yes      | Endpoint ID                 |
| endpointName     | string  | Yes      | Endpoint name               |
| status           | string  | Yes      | Task status                 |
| workerUrl        | string  | Yes      | Worker URL                  |
| webhook          | string  | No       | Webhook URL                 |
| deliveryStatus   | string  | Yes      | Delivery status             |
| deliveryAttempts | integer | Yes      | Number of delivery attempts |
| error            | string  | No       | Error message               |
| input            | object  | Yes      | Task input data             |
| output           | object  | No       | Task output data            |
| headers          | object  | No       | Request headers             |
| createdAt        | integer | Yes      | Creation time (epoch ms)    |
| updatedAt        | integer | Yes      | Update time (epoch ms)      |
| deliveredAt      | integer | No       | Delivery time (epoch ms)    |

#### SubmitTaskRequest

```json
{
  "taskId": "my_task_001",
  "input": { "prompt": "Hello, world!" },
  "workerPort": 8000,
  "processUri": "/v1/chat/completions",
  "webhook": "https://webhook.example.com/callback",
  "webhookAuthKey": "my-secret-key",
  "headers": {
    "Authorization": "Bearer token123"
  }
}
```

| Field          | Type    | Required | Description                                                                        |
| -------------- | ------- | -------- | ---------------------------------------------------------------------------------- |
| taskId         | string  | No       | Task ID (alphanumeric + underscore, max 255 chars). Auto-generated UUID if omitted |
| input          | object  | Yes      | Task input data (any JSON structure)                                               |
| workerPort     | integer | Yes      | Worker port (1-65535)                                                              |
| processUri     | string  | Yes      | Process URI on worker (max 255 chars)                                              |
| webhook        | string  | No       | Webhook URL for async result delivery (max 512 chars)                              |
| webhookAuthKey | string  | No       | Webhook authentication key (max 255 chars)                                         |
| headers        | map     | No       | Request headers to forward with task                                               |

#### TaskStatus

| Status     | Description                 |
| ---------- | --------------------------- |
| PROCESSING | Task is being executed      |
| DELIVERED  | Result delivered to webhook |
| SUCCESS    | Task completed successfully |
| FAILED     | Task execution failed       |

#### DeliveryStatus

| Status                 | Description                     |
| ---------------------- | ------------------------------- |
| INIT                   | Not yet sent                    |
| SUCCESS                | Webhook delivered successfully  |
| FAILED                 | Webhook delivery failed         |
| MAX\_RETRIES\_EXCEEDED | Exceeded maximum retry attempts |

#### WorkerLog

```json
{
  "logs": [
    {
      "timestamp": "2024-01-15T10:30:15.123Z",
      "log": "Starting inference service...",
      "offset": 12345
    }
  ],
  "hasMore": true,
  "nextSearchAfterTime": "1705306215123",
  "nextSearchAfterOffset": "12400"
}
```

| Field                 | Type    | Required | Description                  |
| --------------------- | ------- | -------- | ---------------------------- |
| logs                  | array   | Yes      | Array of log entries         |
| logs\[].timestamp     | string  | Yes      | Log timestamp (ISO 8601)     |
| logs\[].log           | string  | Yes      | Log content                  |
| logs\[].offset        | integer | Yes      | Log offset                   |
| hasMore               | boolean | Yes      | Whether there are more logs  |
| nextSearchAfterTime   | string  | No       | Pagination token (timestamp) |
| nextSearchAfterOffset | string  | No       | Pagination token (offset)    |

***

### Storage Volume Models

#### Volume

```json
{
  "id": 123,
  "name": "my-ceph-volume",
  "sizeInGb": 100,
  "region": "us-west-1",
  "storageType": "CEPH",
  "status": "active",
  "vendorVolumeType": null,
  "mountCount": 1,
  "cost": 5.00,
  "createdAt": 1705306200000
}
```

| Field            | Type    | Required | Description                                |
| ---------------- | ------- | -------- | ------------------------------------------ |
| id               | integer | Yes      | Volume ID                                  |
| name             | string  | Yes      | Volume name                                |
| sizeInGb         | integer | No       | Volume size (GB)                           |
| region           | string  | No       | Region code                                |
| storageType      | string  | Yes      | Storage type: `S3`, `CEPH`, `VENDOR`, `R2` |
| status           | string  | Yes      | Volume status                              |
| vendorVolumeType | string  | No       | Vendor volume type: `NVMe` or `HDD`        |
| mountCount       | integer | Yes      | Mount count                                |
| cost             | number  | Yes      | Cost                                       |
| createdAt        | integer | Yes      | Creation time (epoch ms)                   |

#### CreateVolumeRequest

```json
{
  "name": "my-ceph-volume",
  "storageType": "CEPH",
  "region": "us-west-1",
  "sizeInGb": 100
}
```

| Field            | Type    | Required | Description                                         |
| ---------------- | ------- | -------- | --------------------------------------------------- |
| name             | string  | Yes      | Volume name                                         |
| storageType      | string  | Yes      | Storage type                                        |
| region           | string  | No\*     | Region code (required for CEPH/VENDOR/R2)           |
| sizeInGb         | integer | No\*     | Volume size GB (1-10240) (required for CEPH/VENDOR) |
| vendorVolumeType | string  | No\*     | Volume type (required for VENDOR)                   |

#### RenameVolumeRequest

```json
{
  "name": "my-renamed-volume"
}
```

| Field | Type   | Required | Description     |
| ----- | ------ | -------- | --------------- |
| name  | string | Yes      | New volume name |

#### ResizeVolumeRequest

```json
{
  "sizeInGb": 200
}
```

| Field    | Type    | Required | Description                  |
| -------- | ------- | -------- | ---------------------------- |
| sizeInGb | integer | Yes      | New volume size GB (1-10240) |

#### VolumeStatus

| Status   | Description                 |
| -------- | --------------------------- |
| creating | Volume is being provisioned |
| active   | Volume is ready for use     |
| deleting | Volume is being deleted     |
| resizing | Volume is being resized     |
| error    | Volume operation failed     |

#### StorageType

| Type   | Required Fields                          | Description                                           |
| ------ | ---------------------------------------- | ----------------------------------------------------- |
| S3     | name                                     | Unlimited capacity, no region needed                  |
| R2     | name                                     | Cloudflare R2 storage (S3-compatible, no egress fees) |
| CEPH   | name, region, sizeInGb                   | Network storage for pods                              |
| VENDOR | name, region, sizeInGb, vendorVolumeType | Third-party vendor storage (Verda)                    |

***

### Common Nested Models

#### EnvironmentVariable

```json
{
  "key": "API_KEY",
  "value": "secret"
}
```

| Field | Type   | Required | Description                |
| ----- | ------ | -------- | -------------------------- |
| key   | string | Yes      | Environment variable name  |
| value | string | Yes      | Environment variable value |

#### PortExpose

```json
{
  "port": 8080,
  "protocol": "http"
}
```

| Field    | Type    | Required | Description                         |
| -------- | ------- | -------- | ----------------------------------- |
| port     | integer | Yes      | Container port (1-65535)            |
| protocol | string  | Yes      | Protocol type (e.g., "http", "tcp") |

#### PersistentVolume

```json
{
  "volumeId": 123,
  "mountPath": "/mnt/volume",
  "needBackup": false
}
```

| Field      | Type    | Required | Description              |
| ---------- | ------- | -------- | ------------------------ |
| volumeId   | integer | Yes      | Volume ID                |
| mountPath  | string  | Yes      | Mount path               |
| needBackup | boolean | No       | Whether backup is needed |

#### EndpointResource

```json
{
  "region": "us-east-1",
  "gpuType": "NVIDIA_RTX_4090_24G",
  "gpuCount": 1
}
```

| Field    | Type    | Required | Description    |
| -------- | ------- | -------- | -------------- |
| region   | string  | Yes      | Region code    |
| gpuType  | string  | Yes      | GPU type       |
| gpuCount | integer | Yes      | Number of GPUs |

#### EndpointExpose

```json
{
  "port": 8000,
  "protocol": "http"
}
```

| Field    | Type    | Required | Description              |
| -------- | ------- | -------- | ------------------------ |
| port     | integer | Yes      | Container port (1-65535) |
| protocol | string  | Yes      | Protocol type            |

***

### Enumerations

#### ResourceStatus

Common to all resource types:

| Status       | Description  |
| ------------ | ------------ |
| running      | Running      |
| initializing | Initializing |
| stopped      | Stopped      |
| terminated   | Terminated   |

#### GPUType

| Code                        | Display Name  | VRAM   |
| --------------------------- | ------------- | ------ |
| NVIDIA\_RTX\_4090\_24G      | RTX 4090      | 24 GB  |
| NVIDIA\_RTX\_5090\_32G      | RTX 5090      | 32 GB  |
| NVIDIA\_RTX\_A6000\_48G     | RTX A6000     | 48 GB  |
| NVIDIA\_RTX\_6000\_Ada\_48G | RTX 6000 Ada  | 48 GB  |
| NVIDIA\_A100\_PCIe\_40G     | A100 PCIe 40G | 40 GB  |
| NVIDIA\_A100\_40G           | A100 40G      | 40 GB  |
| NVIDIA\_A100\_PCIe\_80G     | A100 PCIe 80G | 80 GB  |
| NVIDIA\_A100\_80G           | A100 80GB     | 80 GB  |
| NVIDIA\_H100\_PCIe\_80G     | H100 PCIe     | 80 GB  |
| NVIDIA\_H100\_80G           | H100          | 80 GB  |
| NVIDIA\_RTX\_PRO\_6000\_96G | RTX PRO 6000  | 96 GB  |
| NVIDIA\_H200\_141G          | H200          | 141 GB |
| NVIDIA\_B200\_180G          | B200          | 180 GB |
| NVIDIA\_B300\_262G          | B300          | 262 GB |
| AWS\_Trainium\_32G          | Trainium1     | 32 GB  |
| AMD\_MI300X\_192G           | MI300X        | 192 GB |

#### ServiceMode

| Mode   | Description                                                     |
| ------ | --------------------------------------------------------------- |
| ALB    | Application Load Balancer - Direct request routing              |
| QUEUE  | Queue mode - Requests queued and processed by available workers |
| CUSTOM | Custom deployment mode                                          |

#### CredentialType

| Type        | Description                       |
| ----------- | --------------------------------- |
| DOCKER\_HUB | Docker Hub                        |
| GCR         | Google Container Registry         |
| ECR         | Amazon Elastic Container Registry |
| ACR         | Azure Container Registry          |
| PRIVATE     | Private registry                  |

#### ImagePublicType

| Type    | Description   |
| ------- | ------------- |
| PUBLIC  | Public image  |
| PRIVATE | Private image |

