---
icon: location-arrow
---

# Queue-based Elastic Deployment Quickstart

**Welcome!** This guide will walk you through setting up and testing YottaLabs' queue-based elastic deployment from start to finish.

***

### What You'll Accomplish

By the end of this guide, you'll have:

* ✅ Built a queue-compatible worker image
* ✅ Deployed it as a **QUEUE** elastic deployment
* ✅ Submitted tasks through the Queue API
* ✅ Monitored task status in real-time
* ✅ Retrieved and verified task results
* ✅ Confirmed a complete end-to-end workflow

***

### &#x20;Understanding the Queue Model

#### How It Works

In **QUEUE** mode, YottaLabs operates with the following flow:

```
Your Client (curl or Python)
   ↓
   Sends: POST /skywalker/tasks/create
   ↓
Yotta Queue System
   ↓
   Sends: HTTP POST with taskData
   ↓
Your Worker Container (Docker)
   ↓
   Receives: FastAPI endpoint (/run)
   ↓
Your handler(job) function
   ↓
   Returns: result
   ↓
Yotta Queue System
   ↓
   Stores: taskResult for retrieval
```

***

### Step 1: Build Your Worker

#### 1.1 Create Your Business Logic

First, let's create `handler.py` — this is where your core logic lives. This is a super easy example of "echoing" what you prompted back to you.

**File: `handler.py`**

```python
def handler(job):
    """
    job format:
    {
        "input": {
            "prompt": "hello"
        }
    }
    """
    job_input = job.get("input", {})
    prompt = job_input.get("prompt", "no prompt provided")
    return f"You said: {prompt}"
```

#### 1.2 Create the HTTP Server

Now let's wrap your handler with FastAPI so YottaLabs can communicate with it.

**File: `server.py`**

```python
from fastapi import FastAPI, Request
from handler import handler

app = FastAPI()

@app.post("/run")
async def run(request: Request):
    job = await request.json()
    result = handler(job)
    return {
        "result": result
    }
```

> **What's happening here?** YottaLabs sends your task data to the `/run` endpoint, and your server passes it to your handler function.

***

### &#x20;Step 2: Package Everything in Docker

#### 2.1 Define Dependencies

**File: `requirements.txt`**

```txt
fastapi
uvicorn
```

#### 2.2 Create Your Dockerfile

**File: `Dockerfile`**

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir fastapi uvicorn

COPY handler.py server.py ./

EXPOSE 8000

CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### 2.3 Build and Test Locally

**Build your image:**

```bash
docker build -t yotta-queue-test .
```

**Run it locally:**

```bash
docker run -p 8000:8000 yotta-queue-test
```

**Test that it works:**

```bash
curl -X POST http://localhost:8000/run \
  -H "Content-Type: application/json" \
  -d '{"input":{"prompt":"Hello Queue"}}'
```

**You should see:**

```json
{"result":"You said: Hello Queue"}
```

✅ **Great!** Your worker is ready for deployment.

Push the image to Dockerhub for future use. Create,for example, `yotta-queue-test:latest` as my image name and tag.

***

### Step 3: Deploy to YottaLabs

#### Create Your Elastic Deployment

Head to the **YottaLabs Console** and configure:

| Setting          | Value                                   |
| ---------------- | --------------------------------------- |
| **Service Mode** | `QUEUE`                                 |
| **Image**        | `yotta-queue-test` (or your image name) |
| **Worker Port**  | `8000`                                  |
|                  |                                         |

**Wait for deployment to complete** — you'll see **Status = RUNNING** when ready.

#### Save Your Endpoint ID

You'll receive an `Endpoint ID` that looks like this:

```
Endpoint ID: 407712238196040165
```

> ⚠️ **Important**: `Endpoint ID` is NOT `worker ID` shown at the bottom of detail page. Use the code below to get your Endpoint ID list

```bash
curl --request GET \
--url 'https://api.yottalabs.ai/openapi/v1/elastic/deploy/list?statusList=INITIALIZING&statusList=RUNNING&statusList=STOPPED' \
--header 'Content-Type: application/json' \
--header 'X-API-Key: <YOUR_API_KEY>'
```

***

### Step 5: Full Python Integration Test

For a more robust testing workflow, let's use Python.

#### Complete Test Script

**File: `test_remote.py`**

```python
import time
import uuid
import requests
import json

YOTTA_API_BASE = "https://api.yottalabs.ai"

API_KEY = "<YOUR_API_KEY>"
ENDPOINT_ID = "<YOUR_ENDPOINT_ID>"

WORKER_PORT = 8000
PROCESS_URI = "/run"

TIMEOUT_S = 300
POLL_S = 2


def pretty(x):
    return json.dumps(x, ensure_ascii=False, indent=2)


def create_task(prompt):
    url = f"{YOTTA_API_BASE}/openapi/v1/skywalker/tasks/create"
    headers = {
        "X-API-Key": API_KEY,
        "X-Endpoint-ID": ENDPOINT_ID,
        "Content-Type": "application/json",
    }

    user_task_id = f"task_{time.strftime('%Y%m%d_%H%M%S')}_{uuid.uuid4().hex[:8]}"
    payload = {
        "userTaskId": user_task_id,
        "workerPort": WORKER_PORT,
        "processUri": PROCESS_URI,
        "taskData": {"input": {"prompt": prompt}},
    }

    r = requests.post(url, headers=headers, json=payload, timeout=30)
    r.raise_for_status()
    data = r.json()

    if data.get("code") != 10000:
        raise RuntimeError(pretty(data))

    return user_task_id


def get_task(task_id):
    url = f"{YOTTA_API_BASE}/openapi/v1/skywalker/tasks/{task_id}"
    headers = {
        "X-API-Key": API_KEY,
        "X-Endpoint-ID": ENDPOINT_ID,
    }
    r = requests.get(url, headers=headers, timeout=30)
    r.raise_for_status()
    return r.json()["data"]


def wait_done(task_id):
    start = time.time()
    while True:
        detail = get_task(task_id)
        status = detail["status"]

        if status in ("SUCCESS", "FAILED", 2, 3):
            return detail

        if time.time() - start > TIMEOUT_S:
            raise TimeoutError(pretty(detail))

        time.sleep(POLL_S)


def main():
    prompt = "Hello from Queue!"
    task_id = create_task(prompt)

    print("[create]", task_id)

    detail = wait_done(task_id)
    print(pretty(detail))

    assert detail["status"] == "SUCCESS"
    assert detail["taskResult"]["result"] == f"You said: {prompt}"

    print("\n✅ Queue end-to-end test passed!")


if __name__ == "__main__":
    main()
```

#### Run Your Test

```bash
python3 test_remote.py
```

#### Expected Output

```json
{
  "userTaskId": "task_20260129_125952_a4337d96",
  "status": "SUCCESS",
  "workerUrl": "http://localhost:8000/run",
  "taskData": {
    "input": {
      "prompt": "Hello from Queue!"
    }
  },
  "taskResult": {
    "result": "You said: Hello from Queue!"
  },
  "resultSendStatus": "SUCCESS"
}
```

***

### Next Steps

Ready to take things further? Consider exploring:

* **Batch Processing** — Submit multiple tasks at once
* **Monitoring** — Add observability and metrics tracking

***

Need more help? Check out:

* [YottaLabs API Documentation](https://docs.yottalabs.ai/yotta-labs/api-and-sdk/api-spec/elastic-deployment)
