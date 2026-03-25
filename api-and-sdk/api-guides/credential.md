# Credential

#### List All Credentials

```bash
GET /v2/container-registry-auths
```

**Response:**

```json
{
  "message": "success",
  "code": 10000,
  "data": [
    {
      "id": 123,
      "name": "my-docker-registry",
      "type": "DOCKER_HUB",
      "createdAt": "2024-01-15T10:30:00Z"
    }
  ]
}
```

#### Get Credential by ID

```bash
GET /v2/container-registry-auths/{id}
```

**Response:** Same as list

#### Create Credential

```bash
POST /v2/container-registry-auths
```

**Request Body:**

```json
{
  "name": "my-docker-registry",
  "type": "DOCKER_HUB",
  "username": "myuser",
  "password": "mypassword"
}
```

| FIELD    | TYPE   | REQUIRED | DESCRIPTION                                  |
| -------- | ------ | -------- | -------------------------------------------- |
| name     | string | Yes      | Credential name                              |
| type     | string | Yes      | `DOCKER_HUB`, `GCR`, `ECR`, `ACR`, `PRIVATE` |
| username | string | Yes      | Registry username                            |
| password | string | Yes      | Registry password/token                      |

**Response:** Returns created credential

#### Update Credential (Partial)

```bash
PATCH /v2/container-registry-auths/{id}
```

**Request Body:** All fields optional

```json
{
  "name": "updated-name",
  "username": "newuser",
  "password": "newpassword"
}
```

#### Delete Credential

```
DELETE /v2/container-registry-auths/{id}
```

**Note:** Will fail if credential is in use by any Pod or Endpoint.
