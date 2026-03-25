---
hidden: true
---

# Queue-based Elastic Execution

## API List

### Create Task

**Endpoint:** `/openapi/v1/skywalker/tasks/create`&#x20;

**Method:** `POST`&#x20;

**Description:** Creates a new task and submits it to the Elastic Queue. The task is assigned to a worker for execution, and the result is notified asynchronously via callback.

* Idempotent processing is supported based on the `userTaskId`.
* **Rate Limiting:**
  * Per `X-Endpoint-ID`: 40 QPS, 1000 Total Tasks.
  * Global Limit: 200 QPS.

**Authorizations**

| Name          | Location | Type   | Required | Description           |
| ------------- | -------- | ------ | -------- | --------------------- |
| X-API-Key     | header   | string | required | Your API Key          |
| X-Endpoint-ID | header   | string | required | Elastic Deployment ID |

**Request Body (JSON)**

| Name          | Type   | Required | Limits        | Description                                                                                                              |
| ------------- | ------ | -------- | ------------- | ------------------------------------------------------------------------------------------------------------------------ |
| userTaskId    | string | required | Max 255 chars | <p>User-defined custom Task ID.<br>Must only contain letters, numbers, and underscores.</p>                              |
| workerPort    | number | required | 1-65535       | The service port number of the worker.                                                                                   |
| processUri    | string | required | Max 255 chars | <p>The processing interface path on the worker.<br>A leading slash is automatically added.<br>Cannot contain spaces.</p> |
| notifyUrl     | string | optional | Max 512 chars | <p>Callback notification URL for task completion.<br>Must be a valid URL format.</p>                                     |
| notifyAuthKey | string | optional | Max 255 chars | <p>Authentication key for callback notifications.<br>Defined by the user system.</p>                                     |
| taskData      | object | required | -             | <p>Task payload data.<br>Supports any JSON structure (Object, Array, String, etc.).</p>                                  |
| header        | object | optional | -             | <p>Custom HTTP headers forwarded to the worker.<br>Must be a key-value pair structure.</p>                               |

**Example — curl**

```bash
curl --location 'https://api.yottalabs.ai/openapi/v1/skywalker/tasks/create' \
--header 'X-API-Key: <your-api-key>' \
--header 'X-Endpoint-ID: <your-elastic-deployment-id>' \
--header 'Content-Type: application/json' \
--data '{
    "userTaskId": "task_20251111_001",
    "workerPort": 8000,
    "processUri": "/v1/chat/completions",
    "notifyUrl": "http://your-callback-url/notify",
    "notifyAuthKey": "key",
    "taskData": {
        "temperature": 0.5,
        "model": "meta-llama/Llama-3.2-3B-Instruct",
        "messages": [
            {
                "role": "user",
                "content": "Hello"
            }
        ],
        "stream": false
    },
    "header": {
        "Authorization": "Bearer your-token-here",
        "X-Custom-Header": "custom-value"
    }
}'
```

**Response**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
    "userTaskId": "task_20251111_001"
  }
}
```

***

### Get Task Details

**Endpoint:** `/openapi/v1/skywalker/tasks/{userTaskId}`&#x20;

**Method:** `GET`&#x20;

**Description:** Retrieves the details of a specific task.

* **Rate Limiting:**
  * Per `X-Endpoint-ID`: 60 QPS.
  * Global Limit: 400 QPS.

**Authorizations**

| Name          | Location | Type   | Required | Description           |
| ------------- | -------- | ------ | -------- | --------------------- |
| X-API-Key     | header   | string | required | Your API Key          |
| X-Endpoint-ID | header   | string | required | Elastic Deployment ID |

**Path Parameters**

| Name       | Type   | Required | Description  |
| ---------- | ------ | -------- | ------------ |
| userTaskId | string | required | User Task ID |

**Example — curl**

```bash
curl --location 'https://api.yottalabs.ai/openapi/v1/skywalker/tasks/task_20251111_001' \
--header 'X-API-Key: your-api-key-here'
```

**Response**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
    "userTaskId": "task_20251111_001",
    "status": "PROCESSING",
    "workerUrl": "http://localhost:8000/v1/chat/completions",
    "notifyUrl": "http://your-callback-url/notify",
    "createdAt": "2025-11-12 18:02:22.030",
    "updatedAt": "2025-11-12 18:02:22.030",
    "taskData": { ... },
    "header": { ... },
    "resultSendStatus": "INIT",
    "resultSendCount": 0,
    "notifySendCount": 0
  }
}
```

**Response Fields**

| Field              | Type   | Description                                                                                                             |
| ------------------ | ------ | ----------------------------------------------------------------------------------------------------------------------- |
| userTaskId         | string | User Task ID                                                                                                            |
| workerUrl          | string | Full Worker URL (host:port + processUri)                                                                                |
| notifyUrl          | string | Callback notification URL                                                                                               |
| createdAt          | string | Task creation time (ISO 8601 format)                                                                                    |
| updatedAt          | string | Last update time (ISO 8601 format)                                                                                      |
| status             | number | Task Status (0=PROCESSING, 1=DELIVERED, 2=SUCCESS, 3=FAILED)                                                            |
| failedReason       | string | Reason for failure (if the task failed)                                                                                 |
| resultSendStatus   | number | Callback notification status (0=INIT, 1=SUCCESS, 2=FAILED, 3=MAX\_RETRIES\_EXCEEDED)                                    |
| resultSendCount    | number | <p>Number of callback notification attempts.<br>Max 3 retries after initial failure (intervals: 1min, 5min, 15min).</p> |
| nextResultSendTime | string | Next scheduled result transmission time (if retry is needed)                                                            |
| taskData           | object | Original task data payload                                                                                              |
| header             | object | Custom task request headers                                                                                             |
| taskResult         | object | Task execution result                                                                                                   |
| notifySendCount    | number | Total number of notifications sent                                                                                      |
| lastNotifiedAt     | string | Last notification timestamp                                                                                             |

***

### Get Processing Task Count

**Endpoint:** `/openapi/v1/skywalker/tasks/processing/count`&#x20;

**Method:** `GET` **Description:** Retrieves the count of tasks that are currently pending or processing.

* **Rate Limiting:**
  * Per `X-Endpoint-ID`: 60 QPS.
  * Global Limit: 400 QPS.

**Authorizations**

| Name          | Location | Type   | Required | Description           |
| ------------- | -------- | ------ | -------- | --------------------- |
| X-API-Key     | header   | string | required | Your API Key          |
| X-Endpoint-ID | header   | string | required | Elastic Deployment ID |

**Example — curl**

```bash
curl --location 'https://api.yottalabs.ai/openapi/v1/skywalker/tasks/processing/count' \
--header 'X-API-Key: your-api-key-here' \
--header 'X-Endpoint-ID: 378570724762587238' 
```

**Response**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
    "processingCount": 515
  }
}
```

**Response Fields**

| Field           | Type   | Description                               |
| --------------- | ------ | ----------------------------------------- |
| processingCount | number | Number of tasks currently being processed |

***

### List Tasks

**Endpoint:** `/openapi/v1/skywalker/tasks`&#x20;

**Method:** `GET`&#x20;

**Description:** Retrieves a paginated list of tasks.

* **Rate Limiting:**
  * Per `X-Endpoint-ID`: 60 QPS.
  * Global Limit: 400 QPS.

**Authorizations**

| Name          | Location | Type   | Required | Description           |
| ------------- | -------- | ------ | -------- | --------------------- |
| X-API-Key     | header   | string | required | Your API Key          |
| X-Endpoint-ID | header   | string | required | Elastic Deployment ID |

**Query Parameters**

| Name     | Type   | Required | Default | Description                                                                     |
| -------- | ------ | -------- | ------- | ------------------------------------------------------------------------------- |
| status   | number | optional | -       | <p>Filter by Task Status:<br>0=PROCESSING, 1=DELIVERED, 2=SUCCESS, 3=FAILED</p> |
| page     | number | optional | 1       | Page number, starting from 1                                                    |
| pageSize | number | optional | 10      | Number of records per page                                                      |

**Example — curl**

```bash
curl --location 'https://api.yottalabs.ai/openapi/v1/skywalker/tasks?status=1&page=1&pageSize=5' \
--header 'X-API-Key: your-api-key-here'
```

**Response**

Returns an object containing `data` with `items` (TaskDetail\[]) and `pagination` (Pagination).

```json
{
  "message": "success",
  "code": 10000,
  "data": {
    "items": [
      {
        "userTaskId": "task_20251111_001",
        "status": "PROCESSING",
        "workerUrl": "http://localhost:8000/v1/chat/completions",
        ...
      }
    ],
    "pagination": {
      "page": 1,
      "pageSize": 5,
      "totalCount": 515,
      "totalPages": 103
    }
  }
}
```

**Structure Definitions**

**TaskDetail**

| Name             | Type   | Description                                                                          |
| ---------------- | ------ | ------------------------------------------------------------------------------------ |
| userTaskId       | string | User Task ID                                                                         |
| workerUrl        | string | Full Worker URL (host:port + processUri)                                             |
| notifyUrl        | string | Callback notification URL                                                            |
| createdAt        | string | Task creation time (ISO 8601 format)                                                 |
| updatedAt        | string | Last update time (ISO 8601 format)                                                   |
| status           | number | Task Status (0=PROCESSING, 1=DELIVERED, 2=SUCCESS, 3=FAILED)                         |
| resultSendStatus | number | Callback notification status (0=INIT, 1=SUCCESS, 2=FAILED, 3=MAX\_RETRIES\_EXCEEDED) |
| resultSendCount  | number | Number of callback attempts                                                          |
| notifySendCount  | number | Total number of notifications sent                                                   |

**Pagination**

| Name       | Type   | Description                |
| ---------- | ------ | -------------------------- |
| page       | number | Current page number        |
| pageSize   | number | Number of records per page |
| totalCount | number | Total number of records    |
| totalPages | number | Total number of pages      |

***

### Callback Notification

**Endpoint:** `{userNotifyUrl}`&#x20;

**Method:** `POST`&#x20;

**Description:** Task completion result notification (**Caller:** Skywalker → User Application). Skywalker actively calls the `notifyUrl` provided by the user during endpoint creation to notify the result.

**Retry Mechanism:**

* If the initial notification fails (returns a non-2xx status code), the system treats it as a failure and queues a compensation task.
* Retry intervals: 1 minute, 5 minutes, 15 minutes.
* Supports multiple retries until success or the maximum of 3 retries is reached.

**Authorizations**

| Name      | Location | Type   | Required | Description                                                                           |
| --------- | -------- | ------ | -------- | ------------------------------------------------------------------------------------- |
| X-API-Key | header   | string | required | Uses the `notifyAuthKey` parameter from the Create Task interface for authentication. |

**Request Body (JSON)**

| Name           | Type    | Required | Description                                                                                      |
| -------------- | ------- | -------- | ------------------------------------------------------------------------------------------------ |
| task\_id       | number  | required | System Internal Task ID. Used for backward compatibility.                                        |
| user\_task\_id | string  | required | User-defined Custom Task ID. Main identifier (Max 255 chars).                                    |
| endpoint\_id   | string  | required | Endpoint ID identifying where the task belongs (Max 255 chars).                                  |
| status         | string  | required | Task status. Enum values: `Success` or `Failed`.                                                 |
| success        | boolean | required | Task success flag. `true` indicates success, `false` indicates failure.                          |
| timestamp      | string  | required | Notification timestamp (UTC). Format: `YYYY-MM-DD HH:mm:ss.SSS`                                  |
| retry\_count   | number  | required | Retry count. Starts at 1 (initial send is 1).                                                    |
| result         | object  | optional | Task execution result data. Contains the processing result returned by the worker if successful. |
| failed\_reason | string  | optional | Reason for failure. Only exists when `status` is `Failed`.                                       |

**Example — curl**

```json
{
    "endpoint_id": 379017551614185902,
    "status": "SUCCESS",
    "success": true,
    "task_id": 246634865268150272,
    "timestamp": "2025-11-11 15:52:28.385",
    "user_task_id": "task_427",
    "retry_count": 1
}
```

***

## Enums

**Task.status**

| Value | Name       | Description                           |
| ----- | ---------- | ------------------------------------- |
| 0     | PROCESSING | Task is currently being processed     |
| 1     | DELIVERED  | Task has been delivered to the worker |
| 2     | SUCCESS    | Task executed successfully            |
| 3     | FAILED     | Task execution failed                 |

**Task.resultSendStatus**

| Value | Name                   | Description                               |
| ----- | ---------------------- | ----------------------------------------- |
| 0     | INIT                   | Initial state, not sent                   |
| 1     | SUCCESS                | Result sent successfully                  |
| 2     | FAILED                 | Result sending failed                     |
| 3     | MAX\_RETRIES\_EXCEEDED | Exceeded maximum retry attempts (3 times) |

***

## Response Codes

| Code  | Message                                   | Description                                         |
| ----- | ----------------------------------------- | --------------------------------------------------- |
| 10000 | success                                   | Request successful                                  |
| 429   | too many requests                         | Request QPS exceeded rate limit threshold           |
| 40001 | userTaskId is required                    | `userTaskId` field is missing                       |
| 40001 | userTaskId exceeds maximum length...      | `userTaskId` exceeds length limit of 255 characters |
| 40001 | userTaskId can only contain...            | `userTaskId` contains invalid characters            |
| 40001 | workerPort must be at least 1             | Port number below minimum value                     |
| 40001 | workerPort must not exceed 65535          | Port number exceeds maximum value                   |
| 40001 | processUri is required                    | `processUri` field is missing                       |
| 40001 | processUri must not contain spaces        | `processUri` contains spaces                        |
| 40001 | processUri contains invalid characters... | `processUri` contains invalid URI characters        |
| 40001 | notifyUrl must be a valid URL             | `notifyUrl` format is incorrect                     |
| 40001 | taskData is required                      | `taskData` field is missing                         |
| 40001 | taskData cannot be empty                  | `taskData` is empty                                 |
| 40001 | header must be a kv structure             | `header` is not a Key-Value structure               |
| 40101 | Authentication context not found          | Authentication context missing                      |
| 40001 | status must be between 0 and 3            | Task status enum value error                        |
| 40402 | Task not found                            | Task does not exist                                 |
| 50001 | Failed to create a task                   | Internal server error creating task                 |
