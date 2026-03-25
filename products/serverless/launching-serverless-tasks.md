# Launching Serverless Tasks

### ALB Mode

* Navigate to **Compute → Elastic Deployment** in the left menu.
* Click the **Deploy** button in the top-right corner to start the deployment wizard.

<figure><img src="../../.gitbook/assets/image (191).png" alt=""><figcaption></figcaption></figure>

* In the configuration form:
  * Fill in all required fields (marked with \*)
  * Optionally set additional parameters for custom needs
  * Click **Add GPU** to select the GPU type for your workload

{% hint style="warning" %}
**You should put your service port in the HTTP Port field. The worker will be in Running status only when the port is ready.**
{% endhint %}

<figure><img src="../../.gitbook/assets/image (193).png" alt="" width="375"><figcaption></figcaption></figure>

{% hint style="success" %}
**Recommendation:** choose GPUs from multiple regions to maximize redundancy and disaster recovery
{% endhint %}

<figure><img src="../../.gitbook/assets/image (194).png" alt="" width="375"><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (195).png" alt="" width="375"><figcaption></figcaption></figure>

* Click **Deploy** to launch your serverless task.

<figure><img src="../../.gitbook/assets/image (196).png" alt=""><figcaption></figcaption></figure>

### Queue Mode

{% hint style="warning" %}
You should put the port used for your service health check  in the HTTP Port field. The worker will be in Running status only when the port is ready.
{% endhint %}

<figure><img src="../../.gitbook/assets/image (197).png" alt="" width="375"><figcaption></figcaption></figure>

### Custom Mode

Similarly to the ALB mode and Queue mode, you can choose Custom mode to hook up your serverless task with your own infrastructure (e.g. API Gateway, Queue)

<figure><img src="../../.gitbook/assets/image (198).png" alt="" width="375"><figcaption></figcaption></figure>
