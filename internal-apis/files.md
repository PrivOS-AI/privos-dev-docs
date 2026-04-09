# Internal Files API

## Overview

Manage files in MinIO object storage without room-scoped authentication. Provides presigned URLs for direct uploads/downloads, file listing with manifest, file deletion, and automatic task processing (embedding/Weaviate cleanup).

Supports multiple storage roots via the `root` parameter:

| `root` value | MinIO prefix | Use case |
|---|---|---|
| _(empty/omitted)_ | `roomId/` | General room files |
| `markdown` | `markdown/roomId/` | Markdown editor files |

**Base Path:** `/api/v1/internal/files`

## Authentication

All endpoints require:
- `x-api-key`: Internal API key

No room-scoped JWT token is needed (unlike the room-scoped variant).

---

## Endpoints

### Get File Manifest

```http
GET /api/v1/internal/files.manifest
```

**Description:** List all files stored in MinIO for a room with presigned download URLs. Returns files recursively, skipping directory markers.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID to list files for |
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
curl -X GET "https://your-domain.com/api/v1/internal/files.manifest?roomId=ROOM_ID" \
  -H "x-api-key: YOUR_API_KEY"

# List files in a subfolder
curl -X GET "https://your-domain.com/api/v1/internal/files.manifest?roomId=ROOM_ID&folder=Documents/" \
  -H "x-api-key: YOUR_API_KEY"

# List markdown files
curl -X GET "https://your-domain.com/api/v1/internal/files.manifest?roomId=ROOM_ID&root=markdown" \
  -H "x-api-key: YOUR_API_KEY"
```

---

### Generate Upload URL

```http
POST /api/v1/internal/files.uploadUrl
```

**Description:** Generate a presigned PUT URL for direct-to-MinIO upload. Automatically triggers a file upload task for embedding/processing.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID for file storage |
| `key` | string | Yes | File path, relative (e.g., `Documents/report.pdf`) or absolute (e.g., `ROOM_ID/Documents/report.pdf`) |
| `filename` | string | No | Display filename for the upload task (defaults to `key`) |
| `fileId` | string | No | File ID for the upload task (defaults to resolved object key) |
| `root` | string | No | Storage root: `markdown` for `markdown/roomId/` prefix |

**Response:**

```json
{
  "success": true,
  "status": "success",
  "url": "https://minio.example.com/bucket/ROOM_ID/Documents/report.pdf?X-Amz-Algorithm=...&X-Amz-Signature=...",
  "key": "ROOM_ID/Documents/report.pdf",
  "exists": false,
  "task": {
    "success": true,
    "message": "Task created successfully"
  }
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `url` | string | Presigned PUT URL valid for 7 days |
| `key` | string | Full MinIO object key (auto-prepends root + `roomId/` if needed) |
| `exists` | boolean | Whether the file already exists in storage |
| `task.success` | boolean | Whether the upload task was sent successfully |
| `task.message` | string | Task status message |

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
curl -X POST "https://your-domain.com/api/v1/internal/files.uploadUrl" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "roomId": "ROOM_ID",
    "key": "Documents/report.pdf",
    "filename": "report.pdf"
  }'

# Upload to markdown root
curl -X POST "https://your-domain.com/api/v1/internal/files.uploadUrl" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "roomId": "ROOM_ID",
    "key": "doc.md",
    "root": "markdown"
  }'
```

---

### Delete File

```http
DELETE /api/v1/internal/files.byPath
```

**Description:** Delete a file from MinIO by its object key. Validates that the key belongs to the specified room. Automatically triggers a file deletion task for Weaviate cleanup.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID the file belongs to |
| `key` | string | Yes | File path (relative or full object key) |
| `root` | string | No | Storage root: `markdown` for `markdown/roomId/` prefix |
| `filename` | string | No | Display filename for the deletion task (defaults to `key`) |

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
curl -X DELETE "https://your-domain.com/api/v1/internal/files.byPath?roomId=ROOM_ID&key=Documents/report.pdf" \
  -H "x-api-key: YOUR_API_KEY"

# Delete from markdown root
curl -X DELETE "https://your-domain.com/api/v1/internal/files.byPath?roomId=ROOM_ID&key=doc.md&root=markdown" \
  -H "x-api-key: YOUR_API_KEY"

# Delete with filename for task tracking
curl -X DELETE "https://your-domain.com/api/v1/internal/files.byPath?roomId=ROOM_ID&key=Documents/report.pdf&filename=report.pdf" \
  -H "x-api-key: YOUR_API_KEY"
```

---

### Move/Rename File

```http
PUT /api/v1/internal/files.move
```

**Description:** Move or rename a file in MinIO within the same room. Performs a copy-then-delete operation. If a file already exists at the destination, it will be replaced.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID the file belongs to |
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
curl -X PUT "https://your-domain.com/api/v1/internal/files.move" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "roomId": "ROOM_ID",
    "sourceKey": "Documents/report.pdf",
    "destinationKey": "Archives/report.pdf"
  }'

# Rename a file in place
curl -X PUT "https://your-domain.com/api/v1/internal/files.move" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "roomId": "ROOM_ID",
    "sourceKey": "Documents/report.pdf",
    "destinationKey": "Documents/report-final.pdf"
  }'

# Move within markdown root
curl -X PUT "https://your-domain.com/api/v1/internal/files.move" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "roomId": "ROOM_ID",
    "sourceKey": "draft.md",
    "destinationKey": "published/draft.md",
    "root": "markdown"
  }'
```

---

### Sync File Changes

```http
POST /api/v1/internal/files.sync
```

**Description:** Synchronize file/folder changes from MinIO to the database. Called by external services (AI Service, MCP Tools, etc.) after writing files directly to MinIO, so the database catalog stays in sync with MinIO state.

For full documentation with all examples and edge cases, see [Room-Scoped File Sync API](../room-scoped-apis/file-sync.md).

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID that owns the files |
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
curl -X POST "https://your-domain.com/api/v1/internal/files.sync" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "roomId": "ROOM_ID",
    "changes": [
      { "path": "Agent Artifacts/session1/output/", "action": "created" },
      { "path": "Agent Artifacts/session1/output/report.md", "action": "created" },
      { "path": "Agent Artifacts/session1/draft.md", "action": "updated" },
      { "path": "Agent Artifacts/session1/old-output/", "action": "deleted" }
    ]
  }'
```

**Note:** This variant requires `roomId` in the request body. The room-scoped variant (`POST /api/v1/internal/rooms/:roomId/files/sync`) takes `roomId` from the URL path instead.

---

## Task Processing

Both upload and delete operations automatically trigger background tasks:

| Operation | Task Type | Purpose |
|-----------|-----------|---------|
| `files.uploadUrl` | `addFileUploadTask` | Sends file for embedding/processing (Weaviate indexing) |
| `files.byPath` (DELETE) | `addFileDeletionTask` | Cleans up embeddings from Weaviate |

Task failures are logged as warnings but do not fail the API request. The primary MinIO operation (URL generation or file deletion) always completes regardless of task status.

---

## Error Codes

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `error-invalid-params` | 400 | Missing or invalid parameters |
| `error-manifest-failed` | 400 | Failed to list files from MinIO |
| `error-upload-url-failed` | 400 | Failed to generate presigned upload URL |
| `error-forbidden` | 400 | Object key does not belong to this room |
| `error-file-not-found` | 400 | File does not exist in storage |
| `error-delete-failed` | 400 | Failed to delete file from MinIO |
| `error-move-failed` | 400 | Failed to move/rename file in MinIO |
| `error-sync-failed` | 400 | Failed to sync file changes to database |

---

## Storage Structure

Files are stored in MinIO with room-based prefixing:

```
bucket/
├── ROOM_ID_1/                    <- default root
│   ├── Documents/
│   │   ├── report.pdf
│   │   └── notes.md
│   └── Images/
│       └── photo.png
├── markdown/
│   ├── ROOM_ID_1/                <- root=markdown
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
