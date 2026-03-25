# Queue-based

### Key Features

<figure><img src="../../.gitbook/assets/image-20260113175942407.png" alt=""><figcaption></figcaption></figure>

#### 🚀 Elastic Scaling

Queue Mode automatically adjusts the number of active workers based on incoming request volume. When queue load increases, additional workers are provisioned; when demand decreases, excess workers are automatically terminated to save costs.

#### 💰 Transparent Pricing

* **Per-second billing**: Pay only for actual compute time use.
* **No idle charges**: Workers only incur costs while actively processing requests

<figure><img src="../../.gitbook/assets/image-20260113171057414.png" alt=""><figcaption></figcaption></figure>

For more details in price and billing, see [Pricing & Billing | Yotta Labs](https://docs.yottalabs.ai/yotta-labs/products/elastic-deployment/pricing-and-billing)

### Architecture

#### Queue System

The intelligent queue sits at the center of the architecture, managing:

* Request buffering during traffic spikes
* Load distribution across available workers
* Health checks and automatic failover
* Priority-based request handling

#### Worker Management

* **On-demand provisioning**: Workers spin up in seconds when needed
* **Resource isolation**: Each worker operates in a dedicated, secure environment
* **Status monitoring**: Real-time visibility into worker health and performance

### Getting Started

**1. Configure Your Container**

See [Launching a Deployment | Yotta Labs](https://docs.yottalabs.ai/yotta-labs/products/elastic-deployment/launching-a-deployment)

**2.View/Edit/Clone Configurations**

<figure><img src="../../.gitbook/assets/image-20260113171401319.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image-20260113171714059.png" alt=""><figcaption></figcaption></figure>

It would guide you back to configuration setting page. Try cloning or editing by clicking buttons at the bottom (editing is only available when paused/terminated).

<figure><img src="../../.gitbook/assets/image-20260113171806082.png" alt=""><figcaption></figcaption></figure>

**3.Scale Workers**

<figure><img src="../../.gitbook/assets/image-20260113172138544.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image-20260113172201722.png" alt=""><figcaption></figcaption></figure>

❗️After you scale workers, price would change accordingly.

**4.Terminate/Run Deployment**

* Terminate

<figure><img src="../../.gitbook/assets/image-20260113172526963.png" alt=""><figcaption></figcaption></figure>

❗️Every time you click `pause` to terminate, the original service would stop. Once restarted, new worker IDs will be assigned, and uptime will reset, counting from zero again.If no volume is mounted, all temporary files and caches will be lost. Resuming the elastic deployment will require reloading the container image and re-downloading the model.

* Run

<figure><img src="../../.gitbook/assets/image-20260113173935957.png" alt=""><figcaption></figcaption></figure>

**5.Terminate Worker /See log**

<figure><img src="../../.gitbook/assets/image-20260113175411558.png" alt=""><figcaption></figcaption></figure>

:exclamation:When there is only one worker in your configuration, you cannot stop any worker using the terminate button shown above. If you'd like to stop it, please pause the deployment.
