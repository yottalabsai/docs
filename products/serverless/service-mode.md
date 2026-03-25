# Service Mode

We provide **three Service Modes** for Elastic Deployment:

### &#x20;**ALB**

<figure><img src="../../.gitbook/assets/Screenshot 2025-11-25 at 16.01.24.png" alt=""><figcaption></figcaption></figure>



* The system automatically allocates a dedicated **Gateway (ALB)** for this deployment type, responsible for routing external requests to multiple Worker instances based on defined policies, enabling high availability and high-concurrency access.
* The Gateway supports **load balancing, rate limiting, and authentication**, and only forwards **HTTP** requests.
* Users can securely access their deployed services through the unified ALB entry point, without needing to manage the number or distribution of underlying Workers.



#### **Queue**

<figure><img src="../../.gitbook/assets/image (98).png" alt=""><figcaption></figcaption></figure>

* The Queue mode provides **high-availability, scalable asynchronous task processing**.
* Built on a microservices architecture, it supports dynamic scaling and is ideal for workloads that require processing a large volume of async tasks.
* Based on a reliable message-queue mechanism, Queue ensures **stability and durability** during task transmission and execution.
* Once a task is completed, the system automatically sends the results back to your service (**UserBackend**) via callback.

#### **Custom**

* The system directly deploys **Workers** without exposing any external interfaces.
* Services and tasks running inside the Workers are **not directly accessible** to the user.
