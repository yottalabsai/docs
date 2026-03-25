# Pricing & Billing

Elastic Deployment charges are calculated **hourly**, based on the actual compute and storage resources consumed.

$$
Total\ Cost= \sum(GPU_{rate} \times GPU_{count} + Storage_{rate}\times Storage_{size})
$$

#### Billing Formula

Each hour’s cost includes:

* GPU usage cost
* Storage cost

Note: We don't have extra charge for network bandwidth

#### Example

If a user deploys the following resources:

* **2 Workers**, each configured with:
  * GPU: H100 × 2
  * Disk: 100 GB × 2
* H100 GPU unit price: **$1.85 / GPU / hour**
* Disk unit price: **$0.001 / GB / hour**

**Total hourly cost = (2 × 2 × $1.85) + (200 × $0.001 + 200 × $0.001) = $7.4 + $0.4= $7.8 / hour**

#### Billing Characteristics

* **Pay-as-you-go:** Billing stops immediately once resources are released.
* **Unified multi-region billing:** The system aggregates resource usage across all regions.
* **Transparent reporting:** Detailed per-worker billing breakdowns are available in the **Details** page of each **Elastic Deployment**.

#### Deductions and Balance Policy

* Charges are deducted as soon as an Elastic Deployment starts running.
* When your **account balance approaches $0**, all running Elastic Deployments will be automatically terminated.

#### Viewing Billing Details

* Navigate to **Billing** from the left menu.
* The Billing page displays all Elastic Deployment usage and cost details, including per-worker hourly charges and historical summaries.

<figure><img src="../../.gitbook/assets/image (190).png" alt=""><figcaption></figcaption></figure>

