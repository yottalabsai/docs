---
icon: database
---

# Storage

Storage on the platform is divided into 2 types:

1.Container storage&#x20;

2.Persistent Storage

### **Container Storage**

It provides high-speed, local disk space for your running instances. Unlike Persistent Storage, this storage is physically attached to the host machine, offering maximum I/O performance for temporary data.

You can adjust the storage size using the slider or by entering a specific value in the input field.

* **Default Free Tier:** 256 GB.
* **Extra** container storage is charged $0.00005/GB/hr.
* **Scaling:** Move the slider to the right if your dataset or application requires more local disk space. Or just type in a number in the box above.

<figure><img src="../.gitbook/assets/image (118).png" alt="" width="330"><figcaption></figcaption></figure>

:exclamation:**Reminder: This is temporary storage. Data will be erased after the pod is stopped or terminated.**

### **Persistent Storage**

Check this box to enable external storage mounting for your workspace.

<figure><img src="../.gitbook/assets/image (117).png" alt="" width="344"><figcaption></figcaption></figure>

* **Storage Type:**
  * Ceph: Best for high-performance block/file storage and standard file system operations.
  * S3: Ideal for large-scale object storage (AWS S3 or S3-compatible), typically used for datasets or model weights.
* **Resource Selector** : Click to select the specific storage bucket or volume you wish to attach. If you don't have one yet, hit "create".

<figure><img src="../.gitbook/assets/image (114).png" alt="" width="340"><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (115).png" alt="" width="325"><figcaption></figcaption></figure>

* **Mount Path**: Define the absolute path where the storage will be accessible inside your container.
  * _Example:_ `/home/user/data`&#x20;

#### How to Configure

1. Enable the Persistent Storage checkbox.
2. Select your preferred storage protocol: Ceph or S3.
3. Choose the target resource from the dropdown menu.
4. Enter the desired internal directory in the Mount Path field.

###
