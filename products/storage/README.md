---
icon: database
---

# Storage

### Buckets vs. Volumes — Choosing the Right Storage

Both Buckets and Volumes provide persistent storage, but they serve fundamentally different purposes. Use the table below to decide which is right for your use case.

|                                 | **Buckets**                                                                         | **Volumes**                                                                                        |
| ------------------------------- | ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **Underlying protocol**         | S3 / Walrus (object storage)                                                        | Ceph, R2, or Vendor (block / file storage)                                                         |
| **How you access data**         | Via API or HTTP requests                                                            | Mounted as a local directory path inside your container (e.g., `/home/user/data`)                  |
| **Feels like**                  | A cloud drive / network file share accessed through code                            | A hard disk physically attached to your machine                                                    |
| **Best for**                    | Datasets, model weight archives, media outputs, AI workflow artifacts, static files | Active training checkpoints, databases, code, config files, anything requiring frequent read/write |
| **On-chain verification**       | ✅ Every file has a verifiable ID and tracked history on Walrus                      | ❌ Not applicable                                                                                   |
| **File size limit (UI upload)** | Max 20 MB per file, max 500 files per upload                                        | No per-file limit (block device, size set at volume creation)                                      |
| **Capacity model**              | Usage-based (pay for what you store)                                                | Pre-provisioned in GB at creation time (1–10,240 GB)                                               |
| **Typical use cases**           | Training datasets, model releases, generated images/videos/audio, log archives      | In-progress model checkpoints, fine-tuning data staging, application databases, shared code repos  |

**Rule of thumb:**

* If your workload needs to **read or write files rapidly and repeatedly** during a training or inference job → use a **Volume**.
* If you need to **store and share large files** between jobs, archive outputs, or serve data via HTTP → use a **Bucket**.
