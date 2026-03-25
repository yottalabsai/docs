---
icon: person-running
---

# Quickstart

## Get an account

1. Use your email/Google account/Github account to [sign up here](https://console.yottalabs.ai/signup)
2. Verify your email address

## Add a payment method

1. Navigate to the [Billing page](https://console.yottalabs.ai/billing)
2. Choose the credit you'd like to pay
3. Click **Pay now** and add a card
4. You can view the bill on the Billing page.

## Deploy a Pod

Once your account is ready, it’s time to deploy your first Pod:

1. Navigate to the [**Pods** page](https://console.yottalabs.ai/compute/pods) in the web interface.
2. Click the **Deploy** button.
3. From the list of available GPUs, choose **RTX 4090**.
4. In the **Pod Name** field, enter "quickstart"
5. Keep the default settings for **Pod Template**, **GPU Count**, and **Instance Pricing**.
6. Hit **Deploy** to launch your Pod. After a few seconds, you’ll be redirected back to the Pods page.
7. For more guide, check this doc for [GPU pods](gpu-pods.md)

## Explore the Pod

1. **Image Source\&Image:** This section defines the Operating System and Software Stack that will run on your GPU. Docker Hub is the most common option. It pulls pre-built software environments (containers) from the public Docker registry.
2. **Container Storage:** This is the Temporary Workspace (also known as "Root Storage").
   * Size Slider: This is the disk space available for your OS, installed libraries, and temporary files.
   * Cost: 256 GB is free, but extra space costs $0.00005 per GB per hour.
3. **Persistent Storage:** This is your virtual hard drive that survives even if the GPU pod is deleted. Ceph / S3: These are different storage protocols.
   * Ceph usually acts like a normal folder on your machine where you can save data permanently.
   * S3 connects to cloud "buckets" (like AWS S3) for massive datasets.

## Running Code via JupyterLab

1. Return to the Pods page and click **Connect**
2. Choose **Jupyter Lab -> :8888** service. Click on the Arrow Out Box icon.
3. Under the Notebook header, choose the Python 3 (ipykernel) environment.
4. Enter `print("Hello, world!")` into the first cell.
5. Press the Play button (or `Shift + Enter`) to execute. Success! You’ve officially deployed and executed code on your RunPod instance.

## Clean up

1. **Access Pod Settings:** Return to the Pods dashboard and select your active instance.
2. **Delete Operations:** Click the three-dot icon->Terminate.
3. **Confirm Deletion:** Confirmation in the following pop-up.
4. **Suspend Operations:** Click the Pause button.

## What's next?

1. Create [API keys](../api-and-sdk/api-keys.md) to manage your infrastructure through code.
2. Deep dive into the various [pricing policies](billing.md) available for different GPU tiers.
3. Transition to [elastic deployment](serverless/) computing to develop robust, production-grade AI applications.
4. See our [FAQ page](../company/faqs/).
