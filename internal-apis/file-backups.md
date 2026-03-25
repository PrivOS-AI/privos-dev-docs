# File Time Machine API

File version history and restore using MinIO native bucket versioning. Every file upload, edit, or delete automatically creates a new version in MinIO. The Time Machine API exposes these versions for browsing and restoring.

## Prerequisites

MinIO bucket versioning must be enabled:

```bash
mc version enable myminio/privos-dev
mc ilm rule add myminio/privos-dev --noncurrent-expire-days 30
```

## Authentication

All endpoints require user authentication (`authRequired: true`).

- **Read endpoints** (list, download versions): Any room member
- **Restore endpoint**: Room owner or system admin only

---

## Endpoints

### List File Versions

```http
GET /api/v1/files.versions?channelId=CHANNEL_ID&offset=0&count=25
GET /api/v1/files.versions?channelId=CHANNEL_ID&fileId=FILE_ID
```

List MinIO object versions for all files in a channel, or for a specific file. Newest first. Filters out `.folder` placeholder files.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `channelId` | string | Yes | Room ID |
| `fileId` | string | No | Specific file ID (lists only that file's versions) |
| `offset` | number | No | Pagination offset (default: 0) |
| `count` | number | No | Items per page (default: 25) |

**Response:**

```json
{
  "success": true,
  "versions": [
    {
      "name": "CHANNEL_ID/test.md",
      "lastModified": "2026-03-24T13:22:28.757Z",
      "size": 31,
      "versionId": "10e5b7ed-b3e0-4b81-93cb-6537391efbee",
      "isLatest": true,
      "isDeleteMarker": false,
      "etag": "28079018959a833888ee90e5039809dd"
    },
    {
      "name": "CHANNEL_ID/test.md",
      "lastModified": "2026-03-24T12:10:38.941Z",
      "size": 20,
      "versionId": "04569a0b-f231-4577-b8b5-47dfd9841bd3",
      "isLatest": false,
      "isDeleteMarker": false,
      "etag": "aac34d1d751a28b22e1cb571cb348bf6"
    }
  ],
  "count": 2,
  "offset": 0,
  "total": 7
}
```

**Version fields:**

| Field | Description |
|-------|-------------|
| `name` | MinIO object key (channelId/filename) |
| `versionId` | Unique version identifier |
| `isLatest` | `true` for current version |
| `isDeleteMarker` | `true` if file was deleted (old versions still exist) |
| `size` | File size in bytes for this version |
| `lastModified` | When this version was created |

---

### Download File Version

```http
GET /api/v1/files.version.download?filePath=CHANNEL_ID/test.md&versionId=VERSION_ID&channelId=CHANNEL_ID
```

Download a specific historical version of a file as binary.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | Yes | MinIO object key (`name` from versions list) |
| `versionId` | string | Yes | Version ID to download |
| `channelId` | string | No | Room ID (for access validation) |

**Response:** Binary file content with `Content-Disposition: attachment`.

---

### Restore File Version

```http
POST /api/v1/files.version.restore
```

Restore a file to a previous version. Downloads the old version by `versionId` and re-uploads as the current version. Also updates `file_size` in MongoDB. **Room owner or admin only.**

**Body:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `channelId` | string | Yes | Room ID |
| `filePath` | string | Yes | MinIO object key |
| `versionId` | string | Yes | Version ID to restore to |

**Response:**

```json
{
  "success": true,
  "message": "File restored to selected version",
  "restoredVersionId": "04569a0b-f231-4577-b8b5-47dfd9841bd3",
  "newSize": 20
}
```

---

## Admin Endpoints

All admin endpoints require system `admin` role.

### Get Backup Config

```http
GET /api/v1/admin.files.backupConfig?offset=0&count=25&text=search
```

List per-room backup configurations enriched with room names.

**Response:**

```json
{
  "success": true,
  "configs": [
    {
      "channelId": "CHANNEL_ID",
      "roomName": "General",
      "enabled": true,
      "intervalMinutes": 30,
      "retentionDays": 30
    }
  ],
  "count": 1,
  "offset": 0,
  "total": 1
}
```

---

### Update Backup Config

```http
POST /api/v1/admin.files.backupConfig
```

Create or update per-room backup configuration.

**Body:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `channelId` | string | Yes | Room ID |
| `enabled` | boolean | No | Enable/disable for this room |
| `intervalMinutes` | number | No | Display interval (5/15/30/60/360/720/1440) |
| `retentionDays` | number | No | Display retention (7/14/30/90) |

---

## Global Settings

Configurable in **Admin > Settings > Recovery > Files** or via the **Recovery > Files** tab UI.

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `TimeMachine_Enabled` | boolean | `true` | Show Time Machine UI in file management |
| `TimeMachine_Default_Interval` | select | `30` | Default display interval (minutes) |
| `TimeMachine_Retention_Days` | select | `30` | Default display retention period |

**Note:** Actual version retention is controlled by MinIO lifecycle policy (`mc ilm`), not by these settings. The settings control UI display and per-room admin configuration.

## UI Access

- **Time Machine menu** (`...` > clock icon): Visible only to room owners and system admins
- **Recovery > Files tab**: Visible only to system admins (admin panel)
