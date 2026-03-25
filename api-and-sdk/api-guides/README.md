---
description: >-
  Learn how to use Yotta’s APIs to manage GPUs, deploy workloads, and automate
  infrastructure.
icon: book-open
---

# API Guides

## Overview

This API is your gateway to building, scaling, and automating AI workloads on Yotta’s distributed GPU cloud. With just a few API calls, you can spin up compute Pods, query available GPUs, and orchestrate high-performance training or inference workloads seamlessly across regions.



### Base URL

```http
https://api.yottalabs.ai
```

### Authentication

All endpoints require API key authentication via the `X-Api-Key` header:

```
 X-Api-Key: {X-Api-Key}
```

### Standard Response Format

All endpoints return a standard JSON response:

```json
{
  "message": "string",
  "code": 10000,
  "data": { ... }
}
```

| CODE  | DESCRIPTION                |
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

### Common Status Codes

| VM STATUS    | POD STATUS   | ENDPOINT STATUS |
| ------------ | ------------ | --------------- |
| running      | RUNNING      | RUNNING         |
| initializing | INITIALIZING | INITIALIZING    |
| stopped      | STOPPED      | STOPPED         |
| terminated   | TERMINATED   | TERMINATED      |

***

### GPU Types Reference

You can use these codes in your request.

| CODE                        | DISPLAY NAME  | VRAM   |
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

***

### Service Modes (Endpoints)

| MODE   | DESCRIPTION                                                     |
| ------ | --------------------------------------------------------------- |
| ALB    | Application Load Balancer - Direct request routing              |
| QUEUE  | Queue mode - Requests queued and processed by available workers |
| CUSTOM | Custom deployment mode                                          |

### v2 API vs v1 API Changes

| FEATURE       | V1                                             | V2                                               |
| ------------- | ---------------------------------------------- | ------------------------------------------------ |
| Base Path     | `/openapi/v1/...`                              | `/v2/...`                                        |
| Naming        | `podName`, `nickName`                          | `name`                                           |
| Region        | `region` (single)                              | `regionList` (set)                               |
| Registry Auth | `imageRegistryUsername` + `imageRegistryToken` | `containerRegistryAuthId` (credential reference) |
| Timestamps    | `Long` (epoch seconds)                         | `Long` (epoch milliseconds)                      |
| Update Method | `POST /{id}/update`                            | `PATCH /{id}`                                    |
| List Method   | `GET /list`                                    | `GET /`                                          |
| Response      | `data` field may vary                          | Consistent `{message,code,data}`                 |
| Volume Size   | `volumeSizeGb` (integer)                       | `sizeInGb` (integer)                             |
| Storage Type  | Integer (1,2,3,4)                              | String (`S3`,`CEPH`,`VENDOR`,`R2`)               |
| Volume Status | Integer (0,1,2,3,4,5)                          | String (lowercase)                               |

***

### Notes

* All IDs are returned as strings in Endpoint responses for compatibility
* Timestamps are in milliseconds (epoch) for Pods, ISO 8601 for Endpoints
* GPU counts must be powers of 2 (1, 2, 4, 8, ...)
* Minimum container volume is 20 GB for endpoints
* Credentials cannot be retrieved (password/token omitted for security)
* Delete operations will fail if resource is in use
* Volumes with `mountCount > 0` cannot be deleted or resized
* S3/R2 volumes have unlimited size; CEPH/VENDOR volumes require `sizeInGb`
* SYSTEM volumes are not exposed via the v2 API (auto-managed by platform)
* For private images, use `containerRegistryAuthId` to reference stored credentials
* If `imageRegistry` is not specified, Docker Hub is used by default
