# Elastic Deployment

### Elastic Endpoints

#### Create Endpoint

```bash
POST /v2/serverless
```

**Request Body:**

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

| FIELD                   | TYPE    | REQUIRED | DESCRIPTION                                                 |
| ----------------------- | ------- | -------- | ----------------------------------------------------------- |
| name                    | string  | Yes      | Endpoint name (max 20 chars, letters first)                 |
| imageRegistry           | string  | No       | Docker registry URL (default: Docker Hub)                   |
| image                   | string  | Yes      | Docker image                                                |
| containerRegistryAuthId | long    | No       | Container registry credential ID (for private images)       |
| resources               | array   | Yes      | GPU resources                                               |
| workers                 | integer | Yes      | Number of workers                                           |
| containerVolumeInGb     | integer | Yes      | Min 20 GB                                                   |
| environmentVars         | array   | No       | Environment variables                                       |
| expose                  | object  | No       | Port exposure (see below)                                   |
| expose.port             | integer | Yes\*    | Container port (1-65535)                                    |
| expose.protocol         | string  | Yes\*    | Protocol type (e.g., "http", "tcp")                         |
| serviceMode             | string  | Yes      | `ALB`, `QUEUE`, or `CUSTOM`                                 |
| webhook                 | string  | No       | Webhook URL for worker status notifications (max 512 chars) |

\* Required when `expose` is provided.

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
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
}
```

#### Get Endpoint by ID

```bash
GET /v2/serverless/{id}
```

**Response:** Same as create

#### List Endpoints

```bash
GET /v2/serverless
```

**Query Parameters:**

* `statusList` - Filter by status (comma-separated)

#### Update Endpoint

```bash
PATCH /v2/serverless/{id}
```

**Request Body:**

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

| FIELD                 | TYPE    | REQUIRED | DESCRIPTION                                                 |
| --------------------- | ------- | -------- | ----------------------------------------------------------- |
| name                  | string  | Yes      | Endpoint name (max 20 chars, letters first)                 |
| resources             | array   | Yes      | GPU resources                                               |
| workers               | integer | Yes      | Number of workers (min 1)                                   |
| containerVolumeInGb   | integer | Yes      | Min 20 GB                                                   |
| minSingleCardVramInGb | integer | No       | Minimum GPU single card VRAM in GB                          |
| minSingleCardVcpu     | integer | No       | Minimum GPU single card vCPU count                          |
| minSingleCardRamInGb  | integer | No       | Minimum GPU single card RAM in GB                           |
| credentialId          | integer | No       | Container registry credential ID                            |
| initializationCommand | string  | No       | Initialization command                                      |
| environmentVars       | array   | No       | Environment variables                                       |
| expose                | object  | No       | Port exposure (see Section 4.1)                             |
| webhook               | string  | No       | Webhook URL for worker status notifications (max 512 chars) |

#### Endpoint Actions

```bash
POST /v2/serverless/{id}/stop
POST /v2/serverless/{id}/start
DELETE /v2/serverless/{id}
```

#### Scale Workers

```bash
PUT /v2/serverless/{id}/workers?count=4
```

#### List Workers

```bash
GET /v2/serverless/{id}/workers
```

**Query Parameters:**

* `statusList` - Filter by worker status (comma-separated)

#### Tasks API (QUEUE mode)

The Tasks API enables QUEUE-mode endpoints to process asynchronous workloads. Tasks are submitted to an endpoint and processed by available workers.

**Submit Task**

```bash
POST /v2/serverless/{id}/tasks
```

Submit a task to a QUEUE-mode endpoint. The task will be queued and picked up by an available worker.

**Request Body:**

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

| FIELD          | TYPE    | REQUIRED | DESCRIPTION                                                                               |
| -------------- | ------- | -------- | ----------------------------------------------------------------------------------------- |
| taskId         | string  | No       | User-defined task ID (alphanumeric + underscore, max 255). Auto-generated UUID if omitted |
| input          | object  | Yes      | Task input data (any JSON structure)                                                      |
| workerPort     | integer | Yes      | Worker port to forward the task to (1-65535)                                              |
| processUri     | string  | Yes      | Process URI on the worker (max 255 chars)                                                 |
| webhook        | string  | No       | Webhook URL for async result delivery (max 512 chars)                                     |
| webhookAuthKey | string  | No       | Webhook authentication key (max 255 chars)                                                |
| headers        | map     | No       | Headers to forward with the task request                                                  |

**Response:**

```
{
  "message": "success",
  "code": 10000,
  "data": {
    "taskId": "my_task_001"
  }
}
```

**Note:** Only QUEUE-mode endpoints accept task submission. Submitting to a non-QUEUE endpoint returns a parameter error.

**Get Task by ID**

```bash
GET /v2/serverless/{id}/tasks/{taskId}
```

Retrieve full details of a specific task, including input/output data and headers.

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
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
}
```

**List Tasks**

```bash
GET /v2/serverless/{id}/tasks
```

**Query Parameters:**

* `status` - Filter by task status: `PROCESSING`, `DELIVERED`, `SUCCESS`, `FAILED`
* `pageNumber` - Page number (default: 1)
* `pageSize` - Items per page (default: 10)

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
    "items": [
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
        "createdAt": 1705306200000,
        "deliveredAt": 1705306260000,
        "updatedAt": 1705306260000
      }
    ],
    "page": 1,
    "size": 10,
    "total": 42,
    "pages": 5
  }
}
```

**Get Task Count**

```bash
GET /v2/serverless/{id}/tasks/count
```

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
    "processing": 10
  }
}
```

> **Note:** Currently only `processing` count is available. Additional status counts (`total`, `delivered`, `success`, `failed`) will be added in a future release.

**Task Status Values:**

| STATUS     | DESCRIPTION                 |
| ---------- | --------------------------- |
| PROCESSING | Task is being executed      |
| DELIVERED  | Result delivered to webhook |
| SUCCESS    | Task completed successfully |
| FAILED     | Task execution failed       |

**Delivery Status Values:**

| STATUS                 | DESCRIPTION                     |
| ---------------------- | ------------------------------- |
| INIT                   | Not yet sent                    |
| SUCCESS                | Webhook delivered successfully  |
| FAILED                 | Webhook delivery failed         |
| MAX\_RETRIES\_EXCEEDED | Exceeded maximum retry attempts |

**Worker Logs API (Yotta Extension)**

```bash
GET /v2/serverless/{id}/workers/{workerId}/logs
```

Retrieves logs from a specific worker (pod) belonging to an endpoint.

**Query Parameters:**

| PARAMETER         | TYPE    | DEFAULT | MAX  | DESCRIPTION                       |
| ----------------- | ------- | ------- | ---- | --------------------------------- |
| pageSize          | integer | 100     | 1000 | Number of log entries to return   |
| keyword           | string  | -       | -    | Filter logs by keyword            |
| startTime         | string  | -       | -    | Start time (epoch ms or ISO 8601) |
| endTime           | string  | -       | -    | End time (epoch ms or ISO 8601)   |
| searchAfterTime   | string  | -       | -    | Pagination token (timestamp)      |
| searchAfterOffset | long    | -       | -    | Pagination token (offset)         |
| direction         | string  | Forward | -    | "Forward" or "Backward"           |

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
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
}
```

**Notes:**

* This is a Yotta-specific extension for log retrieval
* Supports pagination with `search_after` tokens for efficient log traversal
* Use `direction=Backward` with `nextSearchAfter*` tokens to page through older logs
* Use `direction=Forward` with `nextSearchAfter*` tokens to page through newer logs
* Timestamps support both epoch milliseconds and ISO 8601 format
