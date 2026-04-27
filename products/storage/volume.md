# Volume

### Volumes List Page

<figure><img src="../../.gitbook/assets/image (11).png" alt="" width="386"><figcaption></figcaption></figure>

| Tab                 | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Network Storage** | <p>High-performance Ceph-backed block storage. Pre-provisioned in GB and mounted into Pods or VMs as a local file system. Best for active workloads with frequent read/write operations.<br></p><p><strong>Best for:</strong></p><ul><li>Model training checkpoints written and read repeatedly during a job</li><li>Databases and persistent application state</li><li>Shared scripts, configuration files, and code between sessions</li><li>Any workload requiring low-latency, high-throughput file I/O</li></ul> |
| **Cloud Storage**   | S3-compatible object storage. Usage-based pricing with no fixed capacity. Best for large datasets and model files accessed via API.                                                                                                                                                                                                                                                                                                                                                                                   |
| **System Volume**   | System-managed volumes automatically created by the platform when you create a pod or a virtual machine. **These cannot be manually created or deleted by the user.**                                                                                                                                                                                                                                                                                                                                                 |

#### Table Columns

<figure><img src="../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

| Column              | Description                                                                                                                                                         |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name**            | The unique name of the volume as entered at creation.                                                                                                               |
| **Instance Type**   | The workload type this volume is associated with: `Pod` or `Virtual Machine`.                                                                                       |
| **Size**            | For Network Storage: shows used capacity vs. total provisioned capacity (e.g., `0 B / 256GB`).                                                                      |
| **Cost**            | The accumulated cost incurred by this volume since creation (e.g., `$0.00`).                                                                                        |
| **Monthly Pricing** | The projected monthly cost based on the volume's provisioned size and the applicable rate (e.g., `$6.984` for a 100 GB Network Storage volume at $0.0698/GB/Month). |
| **Volume Type**     | The specific storage variant or hardware type. Displays `–` if not applicable.                                                                                      |
| **Region**          | The geographic region where the volume is provisioned (e.g., `us-east-3`).                                                                                          |
| **Mount Status**    | Indicates whether the volume is currently attached to a running instance. Displays `Unmounted` (in red) when not attached to any workload.                          |

#### + Create Button

<figure><img src="../../.gitbook/assets/image (13).png" alt="" width="152"><figcaption></figcaption></figure>

* **Location:** Top-right corner of the Volumes List page.
* **Function:** Clicking this button reveals a dropdown menu with two options:
  * **Network Storage** — opens the Create Network Storage dialog.
  * **Cloud Storage** — opens the Create Cloud Storage dialog.

> **Note:** System Volumes cannot be manually created. They are provisioned automatically by the platform.

### Creating a Network Storage Volume

Click **+ Create** → **Network Storage** to open the Create Network Storage dialog.

<figure><img src="../../.gitbook/assets/image (14).png" alt="" width="528"><figcaption></figcaption></figure>

| Field        | Required      | Description                                                                                                                                                             |
| ------------ | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name**     | ✅ Yes         | A unique name for this volume.                                                                                                                                          |
| **Workload** | ✅ Yes         | The type of compute instance this volume will be used with. Select `Pod` or `Virtual Machine`. _**This setting determines which instance types can mount the volume.**_ |
| **Region**   | ✅ Yes         | The geographic region where the volume will be provisioned. Available options: `us-east-3`, `us-east-4`.                                                                |
| **Size**     | ✅ Yes         | The pre-provisioned capacity in GB. Enter a numeric value in the input field. The projected monthly cost updates dynamically as you type.                               |
| **Pricing**  | — (read-only) | Displays the rate ($0.0698 / GB / Month) and the projected monthly total based on the size entered.                                                                     |

#### Dialog Buttons

| Button      | Description                                                                                                                               |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Cancel**  | Closes the dialog without creating a volume. No changes are made.                                                                         |
| **Confirm** | Provisions the new Network Storage volume with the specified configuration. The volume will appear in the Network Storage tab once ready. |

***

### Creating a Cloud Storage Volume

Click **+ Create** → **Cloud Storage** to open the Create Cloud Storage dialog.

<figure><img src="../../.gitbook/assets/image (16).png" alt="" width="539"><figcaption></figcaption></figure>

#### Dialog Fields (Cloud Storage)

| Field        | Required      | Description                                                                                                                    |
| ------------ | ------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Name**     | ✅ Yes         | A unique name for this volume. Use lowercase letters, numbers, and hyphens (e.g., `dataset-store`).                            |
| **Workload** | ✅ Yes         | The type of compute instance this volume is intended for. Currently only **Pod** is available.                                 |
| **Pricing**  | — (read-only) | Displays the fixed rate of $0.0493 / GB / Month. No size needs to be configured — you are billed only for actual storage used. |

#### Dialog Buttons

| Button      | Description                                                                                               |
| ----------- | --------------------------------------------------------------------------------------------------------- |
| **Cancel**  | Closes the dialog without creating a volume. No changes are made.                                         |
| **Confirm** | Creates the Cloud Storage volume. It will appear in the Cloud Storage tab, ready to be attached to a Pod. |

***

### Attaching a Volume to a Pod or VM

A volume only becomes active when it is attached to a running compute instance.

You configure this through the **Persistent Storage** setting when launching a Pod or Virtual Machine.

* Navigate to **GPU Pods** or **Virtual Machines** and begin configuring a new instance.

<figure><img src="../../.gitbook/assets/image (17).png" alt="" width="563"><figcaption></figcaption></figure>

* In the launch configuration screen, locate the **Persistent Storage** section and enable the checkbox.

<figure><img src="../../.gitbook/assets/image (18).png" alt="" width="563"><figcaption></figcaption></figure>

* Select the **Storage Type**:
  * Network Storage volumes (block/file storage).
  * Cloud Storage volumes (object storage).
* Set the **Mount Path** — the absolute directory path inside your container where the volume will be accessible (e.g., `/data` or `/workspace/checkpoints`).
* Launch the instance. The volume will be mounted at the specified path and the **Mount Status** in the Volumes list will update accordingly.
