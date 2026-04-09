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

### Move/Rename File

```http
PUT /api/v1/internal/rooms/:roomId/files/move
```

**Description:** Move or rename a file in MinIO within the room. Performs a copy-then-delete operation. If a file already exists at the destination, it will be replaced.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sourceKey` | string | Yes | Source file path (relative or full object key) |
| `destinationKey` | string | Yes | Destination file path (relative or full object key) |
| `root` | string | No | Storage root: `markdown` for `markdown/roomId/` prefix |

**Response:**

```json
{
  "success": true,
  "status": "success",
  "sourceKey": "ROOM_ID/Documents/report.pdf",
  "destinationKey": "ROOM_ID/Archives/report.pdf",
  "replaced": false
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `sourceKey` | string | Full resolved source object key |
| `destinationKey` | string | Full resolved destination object key |
| `replaced` | boolean | Whether an existing file at the destination was replaced |

**Example:**

```bash
# Move file to a different folder
curl -X PUT "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/move" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sourceKey": "Documents/report.pdf",
    "destinationKey": "Archives/report.pdf"
  }'

# Rename a file in place
curl -X PUT "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/move" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sourceKey": "Documents/report.pdf",
    "destinationKey": "Documents/report-final.pdf"
  }'

# Move within markdown root
curl -X PUT "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/move" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sourceKey": "draft.md",
    "destinationKey": "published/draft.md",
    "root": "markdown"
  }'
```

---

### List File Versions

```http
GET /api/v1/fileManagement/versions/:fileId
```

**Description:** List all versions of a file stored in MinIO. Returns version history with editor metadata (user info who made the change).

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fileId` | string | Yes | File key/path in MinIO (e.g., `ROOM_ID/Documents/report.pdf`) |

**Response:**

```json
{
  "success": true,
  "data": {
    "versions": [
      {
        "versionId": "v1234567890",
        "etag": "d41d8cd98f00b204e9800998ecf8427e",
        "size": 102400,
        "lastModified": "2024-01-15T10:30:00.000Z",
        "editor": {
          "userId": "USER_ID",
          "username": "john.doe",
          "name": "John Doe"
        },
        "isCurrent": true
      },
      {
        "versionId": "v1234567800",
        "etag": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
        "size": 98765,
        "lastModified": "2024-01-14T14:20:00.000Z",
        "editor": {
          "userId": "USER_ID_2",
          "username": "jane.smith",
          "name": "Jane Smith"
        },
        "isCurrent": false
      }
    ],
    "fileKey": "ROOM_ID/Documents/report.pdf"
  }
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `versions` | array | Array of file versions sorted by date (newest first) |
| `versionId` | string | MinIO version ID for this version |
| `etag` | string | Entity tag for this version |
| `size` | number | File size in bytes |
| `lastModified` | string | ISO 8601 timestamp when version was created |
| `editor` | object | User who created/uploaded this version |
| `editor.userId` | string | User ID |
| `editor.username` | string | User's login username |
| `editor.name` | string | User's display name |
| `isCurrent` | boolean | Whether this is the current/active version |
| `fileKey` | string | Full file key in MinIO |

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/fileManagement/versions/ROOM_ID%2FDocuments%2Freport.pdf" \
  -H "x-api-key: YOUR_API_KEY"
```

**Notes:**
- `fileId` should be URL-encoded (forward slashes become `%2F`)
- Requires at least `x-api-key` header; room-scoped JWT optional for additional auth
- Returns empty versions array if file has no history

---

### Restore File Version

```http
POST /api/v1/fileManagement/restore/:versionId
```

**Description:** Restore a previous file version as the current version. The old version becomes the new current, and metadata is updated with the restoring user's info.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `versionId` | string | Yes | Version ID to restore (obtained from list versions endpoint) |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fileKey` | string | Yes | Full file key in MinIO (e.g., `ROOM_ID/Documents/report.pdf`) |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Version restored successfully",
    "fileKey": "ROOM_ID/Documents/report.pdf",
    "restoredVersionId": "v1234567800",
    "newCurrentVersion": {
      "versionId": "v1234567800",
      "size": 98765,
      "lastModified": "2024-01-16T09:15:00.000Z",
      "editor": {
        "userId": "CURRENT_USER_ID",
        "username": "current.user",
        "name": "Current User"
      },
      "isCurrent": true
    }
  }
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/fileManagement/restore/v1234567800" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "fileKey": "ROOM_ID/Documents/report.pdf"
  }'
```

**Notes:**
- Restore operation is captured with the current user's metadata
- Original version ID is preserved in version history
- Restoring the current version is a no-op
- All versions remain accessible after restore

---

### Sync File Changes

```http
POST /api/v1/internal/rooms/:roomId/files/sync
```

**Description:** Synchronize file/folder changes from MinIO to the database. Called by external services (AI Service, MCP Tools, etc.) after writing files directly to MinIO, so the database catalog stays in sync with MinIO state.

For full documentation, see [file-sync.md](./file-sync.md).

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `changes` | array | Yes | List of file/folder changes to sync |
| `changes[].path` | string | Yes | Path relative to `{roomId}/` (e.g., `Agent Artifacts/session1/report.md`) |
| `changes[].action` | string | Yes | One of: `created`, `deleted`, `updated` |
| `changes[].type` | string | No | Explicit hint: `file` or `folder`. If omitted, auto-detected via trailing `/` or file extension |

**Path detection:** Trailing `/` or no file extension = folder. Has extension = file.

**Response:**

```json
{
  "status": "success",
  "created": 2,
  "updated": 1,
  "deleted": 1,
  "errors": []
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/sync" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "changes": [
      { "path": "Agent Artifacts/session1/report.md", "action": "created" },
      { "path": "Agent Artifacts/session1/old.md", "action": "deleted" }
    ]
  }'
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
| `error-move-failed` | 400 | Failed to move/rename file in MinIO |
| `error-sync-failed` | 400 | Failed to sync file changes to database |

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
