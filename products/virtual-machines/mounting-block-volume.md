# Mounting Block Volume

#### I. Mounting a Brand New Block Volume

When mounting a brand-new block device (without any data), follow these steps:

**1. Check the Current Block Devices**

Before mounting, confirm the block devices already present on the system.

```
lsblk
```

Use `lsblk` to view all block devices and their mount statuses, for example `/dev/vdb`.

***

**2. Format the Block Device**

Use the code below to format your block device:

```
mkfs.ext4 /dev/<TARGET>
```

* `<TARGET>`: name of the block device, e.g., `vdb`
* Example file system: `ext4`
* ❗️ If the device already contains data, **do not** format it or data would be lost.

***

#### II. Mounting an Existing Block Volume with Data

Whether the device is newly formatted or contains existing data, **the following steps for mounting are the same.**

**1. Verify the Block Device Name**

After successfully mounting, the block device typically appears as something like:

```
/dev/vdb
```

**2. Create a Mount Directory**

Create a directory to mount the disk:

```
mkdir /mnt/<DIR_NAME>
```

* `<DIR_NAME>`: The name of the directory you wish to use, e.g., `data`.

**3. Mount the Block Device**

Mount the block device to the specified directory:

```
mount /dev/<TARGET> /mnt/<DIR_NAME>
```

e.g.

```
mount /dev/vdb /mnt/data
```

***

#### III. (Recommended) Configure Auto-Mount on Boot

To ensure the Block Volume is automatically mounted after system reboots, add it to the `/etc/fstab` file.

**Add fstab Configuration**

```
echo "/dev/<TARGET> /mnt/<DIR_NAME> ext4 defaults,nofail 0 0" >> /etc/fstab
```

e.g.

```
echo "/dev/vdb /mnt/data ext4 defaults,nofail 0 0" >> /etc/fstab
```

| **PARAMETER** | **DESCRIPTION**                                                    |
| ------------- | ------------------------------------------------------------------ |
| **defaults**  | Use default mount parameters                                       |
| **nofail**    | Even if the disk does not exist, it does not affect system startup |
| **0 0**       | Do not perform **dump** and **fsck** checks                        |
