# Pods

#### Create Pod

```bash
POST /v2/pods
```

**Request Body:**

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

| FIELD                   | TYPE    | REQUIRED | DESCRIPTION                                           |
| ----------------------- | ------- | -------- | ----------------------------------------------------- |
| regions                 | set     | No       | Acceptable region codes for scheduling                |
| name                    | string  | Yes      | Pod name (1-255 chars)                                |
| image                   | string  | Yes      | Docker image                                          |
| imageRegistry           | string  | No       | Docker registry URL (default: Docker Hub)             |
| containerRegistryAuthId | long    | No       | Container registry credential ID (for private images) |
| imagePublicType         | string  | No       | Image type: `PUBLIC` or `PRIVATE` (default: `PUBLIC`) |
| resourceType            | string  | No       | Resource type: `GPU` or `CPU` (default: `GPU`)        |
| gpuType                 | string  | Yes      | GPU type (e.g., "NVIDIA\_RTX\_4090\_24G")             |
| gpuCount                | integer | Yes      | Number of GPUs (must be power of 2)                   |
| minSingleCardVramInGb   | integer | No       | Minimum single card VRAM in GB                        |
| minSingleCardRamInGb    | integer | No       | Minimum single card RAM in GB                         |
| minSingleCardVcpu       | integer | No       | Minimum single card vCPU count                        |
| shmInGb                 | integer | No       | Shared memory size in GB                              |
| containerVolumeInGb     | integer | No       | Container volume size in GB                           |
| persistentVolumeInGb    | integer | No       | Persistent volume size in GB                          |
| persistentMountPath     | string  | No       | Persistent volume mount path                          |
| initializationCommand   | string  | No       | Initialization command to run on container start      |
| environmentVars         | array   | No       | Environment variables                                 |
| expose                  | array   | No       | Ports to expose                                       |
| persistentVolumes       | array   | No       | Persistent volumes configuration                      |

**Private Image Authentication:**

For private images, provide either:

1. `containerRegistryAuthId` - Reference to stored credential (recommended)
2. Both `imageRegistryUsername` and `imageRegistryPassword` - Direct credentials (deprecated, use `containerRegistryAuthId` instead)

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
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
}
```

#### Get Pod by ID

```bash
GET /v2/pods/{id}
```

**Response:** Same as create

#### List Pods

```bash
GET /v2/pods
```

**Query Parameters:**

* `regionList` - Filter by regions (comma-separated)
* `statusList` - Filter by status codes (comma-separated)

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": [ ... ]
}
```

#### Delete Pod

```bash
DELETE /v2/pods/{id}
```

#### Pod Actions

```bash
POST /v2/pods/{id}/pause
POST /v2/pods/{id}/resume
```
