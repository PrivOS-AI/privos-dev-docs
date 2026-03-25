# Room-Scoped Files API

## Overview

Manage files in MinIO object storage within a specific room context. Provides presigned URLs for direct browser uploads/downloads, file listing with manifest, and file deletion.

Supports multiple storage roots via the `root` parameter:

| `root` value | MinIO prefix | Use case |
|---|---|---|
| _(empty/omitted)_ | `roomId/` | General room files |
| `markdown` | `markdown/roomId/` | Markdown editor files |

**Base Path:** `/api/v1/internal/rooms/:roomId/files`

## Authentication

All endpoints require:
- `x-api-key`: Internal API key
- `Authorization: Bearer <JWT_TOKEN>`: Room-specific JWT token

---

## Endpoints

### Get File Manifest

```http
GET /api/v1/internal/rooms/:roomId/files/manifest
```

**Description:** List all files stored in MinIO for a room with presigned download URLs. Returns files recursively, skipping directory markers.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `folder` | string | No | Sub-folder within the room (e.g., `Documents/`) |
| `root` | string | No | Storage root: `markdown` for `markdown/roomId/` prefix |

**Response:**

```json
{
  "success": true,
  "status": "success",
  "data": [
    {
      "key": "ROOM_ID/Documents/report.pdf",
      "size": 102400,
      "lastModified": "2024-01-15T10:30:00.000Z",
      "eTag": "d41d8cd98f00b204e9800998ecf8427e",
      "url": "https://minio.example.com/bucket/ROOM_ID/Documents/report.pdf?X-Amz-Algorithm=..."
    }
  ]
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `key` | string | Full MinIO object key (includes root + `roomId/` prefix) |
| `size` | number | File size in bytes |
| `lastModified` | string | ISO 8601 timestamp of last modification |
| `eTag` | string | Entity tag for cache validation |
| `url` | string | Presigned GET URL valid for 7 days |

**Example:**

```bash
# List all files in a room
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/manifest" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"

# List files in a subfolder
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/manifest?folder=Documents/" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"

# List markdown files
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/manifest?root=markdown" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Generate Upload URL

```http
POST /api/v1/internal/rooms/:roomId/files/upload-url
```

**Description:** Generate a presigned PUT URL for direct-to-MinIO upload. The client can then use this URL to upload the file directly without proxying through the server.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `key` | string | Yes | File path, relative (e.g., `Documents/report.pdf`) or absolute (e.g., `ROOM_ID/Documents/report.pdf`) |
| `root` | string | No | Storage root: `markdown` for `markdown/roomId/` prefix |

**Response:**

```json
{
  "success": true,
  "status": "success",
  "url": "https://minio.example.com/bucket/ROOM_ID/Documents/report.pdf?X-Amz-Algorithm=...&X-Amz-Signature=...",
  "key": "ROOM_ID/Documents/report.pdf",
  "exists": false
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `url` | string | Presigned PUT URL valid for 7 days |
| `key` | string | Full MinIO object key (auto-prepends root + `roomId/` if needed) |
| `exists` | boolean | Whether the file already exists in storage |

**Upload with presigned URL:**

```bash
# The returned URL can be used with a PUT request to upload directly
curl -X PUT "<PRESIGNED_URL>" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @/path/to/local/file.pdf
```

**Example:**

```bash
# Upload to room files
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/upload-url" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "Documents/report.pdf"
  }'

# Upload to markdown root
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/upload-url" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "doc.md",
    "root": "markdown"
  }'
```

---

### Delete File

```http
DELETE /api/v1/internal/rooms/:roomId/files/by-path
```

**Description:** Delete a file from MinIO by its object key. Validates that the key belongs to the specified room.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `key` | string | Yes | File path (relative or full object key) |
| `root` | string | No | Storage root: `markdown` for `markdown/roomId/` prefix |

**Response:**

```json
{
  "success": true,
  "status": "success",
  "message": "Deleted: ROOM_ID/Documents/report.pdf"
}
```

**Example:**

```bash
# Delete from room files
curl -X DELETE "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/by-path?key=Documents/report.pdf" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"

# Delete from markdown root
curl -X DELETE "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/by-path?key=doc.md&root=markdown" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

## Error Codes

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `error-manifest-failed` | 400 | Failed to list files from MinIO |
| `error-invalid-params` | 400 | Missing or invalid parameters (e.g., missing `key`) |
| `error-forbidden` | 400 | Object key does not belong to this room |
| `error-file-not-found` | 400 | File does not exist in storage |
| `error-upload-url-failed` | 400 | Failed to generate presigned upload URL |
| `error-delete-failed` | 400 | Failed to delete file from MinIO |

---

## Storage Structure

Files are stored in MinIO with room-based prefixing:

```
bucket/
├── ROOM_ID_1/                    ← default root
│   ├── Documents/
│   │   ├── report.pdf
│   │   └── notes.md
│   └── Images/
│       └── photo.png
├── markdown/
│   ├── ROOM_ID_1/                ← root=markdown
│   │   └── doc.md
│   └── ROOM_ID_2/
│       └── readme.md
├── ROOM_ID_2/
│   └── ...
```

**Key Points:**
- Each room's files are isolated under `{roomId}/` or `{root}/{roomId}/` prefix
- The `root` parameter selects which storage prefix to use (allowed values: `markdown`)
- The manifest endpoint returns all files recursively within the resolved prefix
- Upload keys are auto-prefixed with the resolved room prefix if not already present
- Delete validates that the resolved key belongs to the specified room
- Presigned URLs expire after **7 days** (604800 seconds)
