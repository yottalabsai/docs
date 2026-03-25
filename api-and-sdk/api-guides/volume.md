# Volume

#### Create Volume

```bash
POST /v2/volumes
```

**Request Body:**

```json
{
  "name": "my-ceph-volume",
  "storageType": "CEPH",
  "region": "us-west-1",
  "sizeInGb": 100
}
```

| FIELD            | TYPE    | REQUIRED       | DESCRIPTION                             |
| ---------------- | ------- | -------------- | --------------------------------------- |
| name             | string  | Yes            | Volume name (valid volume name format)  |
| storageType      | string  | Yes            | `S3`, `CEPH`, `VENDOR`, or `R2`         |
| region           | string  | CEPH/VENDOR/R2 | Storage region code (e.g., "us-west-1") |
| sizeInGb         | integer | CEPH/VENDOR    | Volume size in GB (1-10240)             |
| vendorVolumeType | string  | VENDOR         | Volume type: `NVMe` or `HDD`            |

**Storage Type Details:**

| TYPE   | REQUIRED FIELDS                          | NOTES                                                 |
| ------ | ---------------------------------------- | ----------------------------------------------------- |
| S3     | name                                     | Unlimited size, no region needed                      |
| R2     | name                                     | Cloudflare R2 storage (S3-compatible, no egress fees) |
| CEPH   | name, region, sizeInGb                   | Network storage for pods                              |
| VENDOR | name, region, sizeInGb, vendorVolumeType | Third-party vendor storage (Verda)                    |

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
    "id": 123,
    "name": "my-ceph-volume",
    "sizeInGb": 100,
    "region": "us-west-1",
    "storageType": "CEPH",
    "status": "creating",
    "vendorVolumeType": null,
    "mountCount": 0,
    "cost": 0.00,
    "createdAt": 1705306200000
  }
}
```

#### 5.2 List Volumes (Paginated)

```bash
GET /v2/volumes
```

**Query Parameters:**

| PARAMETER   | TYPE   | REQUIRED | DEFAULT | DESCRIPTION                          |
| ----------- | ------ | -------- | ------- | ------------------------------------ |
| page        | long   | No       | 1       | Page number (1-based)                |
| size        | long   | No       | 10      | Page size                            |
| storageType | string | **Yes**  | -       | Storage type: `S3`, `CEPH`, `VENDOR` |
| region      | string | No       | -       | Filter by region code                |

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": {
    "items": [
      {
        "id": 123,
        "name": "my-ceph-volume",
        "sizeInGb": 100,
        "region": "us-west-1",
        "storageType": "CEPH",
        "status": "active",
        "vendorVolumeType": null,
        "mountCount": 1,
        "cost": 5.00,
        "createdAt": 1705306200000
      }
    ],
    "page": 1,
    "size": 10,
    "total": 42,
    "pages": 5
  }
}
```

#### Get Volume by ID

```bash
GET /v2/volumes/{id}
```

**Response:** Same as create response (with current status)

#### Delete Volume

```bash
DELETE /v2/volumes/{id}
```

**Note:** Only volumes with `mountCount=0` can be deleted. Attempting to delete a mounted volume will fail.

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": true
}
```

#### 5.5 Rename Volume

```bash
POST /v2/volumes/{id}/name
```

**Request Body:**

```json
{
  "name": "my-renamed-volume"
}
```

| FIELD | TYPE   | REQUIRED | DESCRIPTION                                |
| ----- | ------ | -------- | ------------------------------------------ |
| name  | string | Yes      | New volume name (valid volume name format) |

**Response:** Returns updated volume (same format as create response)

#### 5.6 Resize Volume

```bash
POST /v2/volumes/{id}/resize
```

Resize a CEPH or VENDOR volume. Only volumes with `ACTIVE` status and `mountCount=0` can be resized.

**Request Body:**

```json
{
  "sizeInGb": 200
}
```

| FIELD    | TYPE    | REQUIRED | DESCRIPTION                     |
| -------- | ------- | -------- | ------------------------------- |
| sizeInGb | integer | Yes      | New volume size in GB (1-10240) |

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": null
}
```

**Volume Status Values:**

| STATUS   | DESCRIPTION                 |
| -------- | --------------------------- |
| creating | Volume is being provisioned |
| active   | Volume is ready for use     |
| deleting | Volume is being deleted     |
| resizing | Volume is being resized     |
| error    | Volume operation failed     |
