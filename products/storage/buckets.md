# Buckets

### Navigating to Buckets

<figure><img src="../../.gitbook/assets/image.png" alt="" width="143"><figcaption></figcaption></figure>

***

### Buckets List Page

After selecting **Buckets** from the navigation, you will land on the Buckets List page. This page displays all existing buckets associated with your account and provides controls for creating new ones.

#### Table Columns

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

| COLUMN             | DESCRIPTION                                                                                 |
| ------------------ | ------------------------------------------------------------------------------------------- |
| **Name**           | The unique name of the bucket.                                                              |
| **Storage Source** | The underlying storage protocol used by this bucket.                                        |
| **Region**         | **Global** indicates the bucket is not restricted to a specific geographic location.        |
| **Size**           | The total amount of data currently stored in the bucket, displayed in human-readable units  |
| **Action**         | Find the **Delete** button here(red trash-can icon).                                        |

> ⚠️ Warning: This action is irreversible. Deleted data cannot be recovered. It is recommended to verify that the bucket is empty before clicking delete.

***

### + Creating a New Bucket

<figure><img src="../../.gitbook/assets/image (2).png" alt="" width="291"><figcaption></figcaption></figure>



Click the **+ Create** button in the top-right corner of the Buckets List page to open the Create dialog.

#### Dialog Fields

| FIELD              | REQUIRED | DESCRIPTION                                                                                                                                         |
| ------------------ | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name**           | ✅ Yes    | A unique identifier for your bucket. Choose a descriptive name that reflects the purpose of the bucket (e.g., `my-project-assets`, `user-uploads`). |
| **Storage Source** | ✅ Yes    | A dropdown for selecting the storage backend. Click to expand the options. Currently, **Walrus** is the only available choice.                      |

***

### Bucket Detail Page

Clicking the name of any bucket in the list opens the **Bucket Detail** page. This page shows all files stored within that specific bucket and provides tools to upload new files.

<figure><img src="../../.gitbook/assets/image (3).png" alt="" width="563"><figcaption></figcaption></figure>

#### Table Columns

<figure><img src="../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

| COLUMN                 | DESCRIPTION                                                                                                                             |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **Name**               | The file name as stored on the Walrus network.                                                                                          |
| **View On Blockchain** | A link that allows you to inspect the file's record on the underlying blockchain, providing on-chain verification of the stored object. |
| **Type**               | The MIME type or file format of the stored file                                                                                         |
| **Last Modified**      | The timestamp indicating when the file was last updated or uploaded.                                                                    |
| **Size**               | The file size in human-readable units.                                                                                                  |
| **Action**             | Controls for managing individual files within the bucket (e.g., delete).                                                                |

***

### Uploading Files

To upload files into a bucket, navigate to the desired bucket by clicking its name, then click the **Upload** button.

<figure><img src="../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

#### Upload Dialog

<figure><img src="../../.gitbook/assets/image (6).png" alt="" width="375"><figcaption></figcaption></figure>

The dialog contains a drag-and-drop area with the following constraints:

| CONSTRAINT               | LIMIT         |
| ------------------------ | ------------- |
| Maximum files per upload | **500 files** |
| Maximum size per file    | **20 MB**     |

### Complete Workflow Summary

The following summarizes the end-to-end flow for using the Buckets feature:

1. Navigate to **Storage > Buckets** using the left sidebar.
2. Review existing buckets in the list table, noting their name, storage source, region, and current size.
3. Click **+ Create** to add a new bucket, providing a name and selecting **Walrus** as the storage source.
4. Click a bucket name to open its detail view.
5. Click **Upload** to open the upload dialog, select or drag in your files, and confirm with **OK**.
6. Use the **View On Blockchain** column to verify file integrity on-chain.
7. Use the **delete icon** (trash can) in the Action column to remove a bucket when it is no longer needed.

***
