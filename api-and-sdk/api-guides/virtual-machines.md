# Virtual Machines

#### Create VM

```bash
POST /v2/vms
```

**Request Body:**

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

| FIELD            | TYPE    | REQUIRED | DESCRIPTION                      |
| ---------------- | ------- | -------- | -------------------------------- |
| vmTypeId         | integer | Yes      | VM type ID                       |
| region           | string  | Yes      | Region code (e.g., "us-east-1")  |
| name             | string  | Yes      | VM display name                  |
| isSpot           | integer | No       | 0=on-demand, 1=spot (default: 0) |
| volumeMountPaths | map     | No       | Volume ID to mount path mapping  |

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
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
}
```

#### Get VM by ID

```bash
GET /v2/vms/{id}
```

**Response:** Same as create

#### List VMs (Paginated)

```bash
GET /v2/vms
```

**Query Parameters:**

| PARAMETER | TYPE   | DEFAULT | DESCRIPTION                        |
| --------- | ------ | ------- | ---------------------------------- |
| page      | long   | 1       | Page number (1-based)              |
| size      | long   | 10      | Page size                          |
| status    | string | -       | Filter by status (e.g., "running") |

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
    "items": [ ... ],
    "page": 1,
    "size": 10,
    "total": 42,
    "pages": 5
  }
}
```

#### Update VM (Rename)

```bash
PATCH /v2/vms/{id}
```

**Request Body:**

```json
{
  "name": "updated-vm-name"
}
```

#### Terminate VM

```bash
DELETE /v2/vms/{id}
```

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": true
}
```

#### Get VM Types

```bash
GET /v2/vms/types
```

Returns all available GPU types and their availability by region. Required for creating VMs - the `vmTypeId` in create request comes from this list.

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": [
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
    },
    {
      "gpuType": "NVIDIA_RTX_A6000_48G",
      "regions": [
        {
          "region": "us-west-1",
          "regionName": "US West",
          "available": true,
          "vmTypeId": 2
        }
      ]
    }
  ]
}
```
