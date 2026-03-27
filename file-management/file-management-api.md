# File Management API Documentation

## Overview

File Management API provides file and folder management within channels. Files are stored on MinIO (S3-compatible) with metadata on MongoDB, supporting:

- Upload/download files (including chunked upload for large files)
- Organize files in folder hierarchy
- Search and filter files/folders
- AI embedding for files
- Duplicate file detection and conflict resolution
- File content editing (markdown, code files)
- MinIO sync and storage management
- Webhook triggers for file/folder events
- Statistics and storage management

**Base URL:** `https://your-domain.com/api/v1`

**Authentication:**
```http
X-Auth-Token: <your_auth_token>
X-User-Id: <your_user_id>
Content-Type: application/json
```

*Note: File upload uses `multipart/form-data`*

---

## Data Models

### File Object
```typescript
{
  _id: string;                    // Unique file identifier
  name: string;                   // Original filename
  folder_id?: string | null;      // Parent folder ID (null = root)
  channel_id: string;             // Channel/Room ID
  file_path?: string;             // MinIO storage path
  file_size?: number;             // File size (bytes)
  file_type?: string;             // File extension (pdf, jpg, etc)
  user_id?: string;               // Uploader ID
  is_embedded: boolean;           // AI embedding status
  embedding_task_id?: string;     // AI embedding task ID
  downloadUrl?: string;           // Presigned download URL (temporary)
  created_at: string;             // ISO 8601 timestamp
  updated_at: string;             // ISO 8601 timestamp
}
```

### Folder Object
```typescript
{
  _id: string;                    // Unique folder identifier
  name: string;                   // Folder name
  father?: string | null;         // Parent folder ID (null = root)
  channel_id: string;             // Channel/Room ID
  folder_path?: string;           // Display path
  user_id?: string;               // Creator ID
  created_at: string;             // ISO 8601 timestamp
  updated_at: string;             // ISO 8601 timestamp
}
```

### Standard Response Format
```typescript
{
  success: boolean;               // true if successful
  data?: any;                     // Response data
  error?: string;                 // Error message if failed
  message?: string;               // Additional message
}
```

### Duplicate Conflict Response
```typescript
{
  success: false,
  error: "DUPLICATE_FILE" | "DUPLICATE_AT_DESTINATION",
  message: string,
  conflict: {
    existingFile?: {              // Existing file info
      _id: string;
      name: string;
      file_size?: number;
      uploadedAt: string;
    };
    itemType?: "file" | "folder"; // Item type when moving
    existingItem?: object;        // Existing item at destination
    movingItem?: object;          // Item being moved
    options: ["replace", "keep_both", "cancel"];
  }
}
```

---

## API Endpoints

### File Operations

#### 1. Upload File (Direct)
```
POST /file-management.files.upload
Content-Type: multipart/form-data
```

Direct file upload (suitable for files below `FileUpload_MaxDirectFileSize` setting, default 100MB).

**Request Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| files | File | Yes | File to upload |
| channelId | string | Yes | Target channel ID |
| folderId | string | No | Target folder ID (omit for root) |
| enableEmbedding | string | No | "true"=enable AI, "false"=disable, omit=server default |

**Response (Success):**
```json
{
  "success": true,
  "message": "File upload successful",
  "file": { /* File Object */ },
  "filename": "document.pdf",
  "channelId": "CHANNEL_ID",
  "fileSize": 1024000
}
```

**Response (Duplicate Detected):**
```json
{
  "success": false,
  "error": "DUPLICATE_FILE",
  "message": "File \"document.pdf\" already exists in this folder",
  "conflict": {
    "existingFile": {
      "_id": "FILE_ID",
      "name": "document.pdf",
      "file_size": 1024000,
      "uploadedAt": "2024-01-15T10:00:00Z"
    },
    "options": ["replace", "keep_both", "cancel"]
  }
}
```

When receiving a duplicate response, resend the request with an additional `duplicateAction` parameter:
- `replace` - Replace the existing file
- `keep_both` - Keep both files (auto-rename with suffix "_1", "_2"...)
- `cancel` - Cancel upload

---

#### 2. Chunked Upload (Large Files >= `FileUpload_MaxDirectFileSize`)

For large files that exceed Cloudflare/direct upload limits. 3-step process:

##### 2a. Initialize Chunked Upload
```
POST /file-management.files.upload-chunked-init
```

**Request Body:**
```json
{
  "fileName": "large-video.mp4",
  "fileSize": 524288000,
  "fileType": "video/mp4",
  "channelId": "CHANNEL_ID",
  "folderId": "FOLDER_ID",
  "duplicateAction": "replace"
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| fileName | string | Yes | File name |
| fileSize | number | Yes | File size (bytes) |
| fileType | string | Yes | MIME type |
| channelId | string | Yes | Channel ID |
| folderId | string | No | Folder ID |
| duplicateAction | string | No | "replace" / "keep_both" / "cancel" |

**Response:**
```json
{
  "success": true,
  "sessionId": "SESSION_ID",
  "chunkSizes": [20971520, 20971520, 10485760],
  "totalChunks": 3,
  "maxDirectSize": 104857600,
  "fileName": "large-video.mp4"
}
```

##### 2b. Upload Chunk
```
POST /file-management.files.upload-chunk
Content-Type: multipart/form-data
```

**Request Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| file | File | Yes | Chunk data |
| sessionId | string | Yes | Session ID from init |
| chunkIndex | number | Yes | Chunk index (0-based) |
| checksum | string | Yes | MD5 checksum of the chunk |

**Response:**
```json
{
  "success": true,
  "chunkIndex": 0,
  "received": true
}
```

##### 2c. Complete Chunked Upload
```
POST /file-management.files.upload-chunked-complete
```

**Request Body:**
```json
{
  "sessionId": "SESSION_ID",
  "channelId": "CHANNEL_ID",
  "folderId": "FOLDER_ID",
  "finalChecksum": "MD5_HASH",
  "enableEmbedding": true
}
```

**Response:**
```json
{
  "success": true,
  "file": { /* File Object */ }
}
```

##### 2d. Cancel Chunked Upload
```
POST /file-management.files.upload-chunked-cancel
```

**Request Body:**
```json
{
  "sessionId": "SESSION_ID"
}
```

---

#### 3. Get Files in Channel
```
GET /file-management.files.channel/:channelId
```

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| folderId | string | - | Filter by folder ID |
| count | number | 50 | Items per page (max 100) |
| offset | number | 0 | Pagination offset |

**Response:**
```json
{
  "success": true,
  "files": [ /* File Object[] */ ],
  "count": 10,
  "offset": 0,
  "total": 100
}
```

---

#### 4. Get File by ID
```
GET /file-management.files/:fileId
```

**Response:**
```json
{
  "success": true,
  "file": { /* File Object */ }
}
```

---

#### 5. Rename File
```
PUT /file-management.files/:fileId/rename
```

**Request Body:**
```json
{
  "name": "new-filename.pdf"
}
```

**Response:**
```json
{
  "success": true,
  "file": { /* File Object */ },
  "message": "File renamed successfully"
}
```

---

#### 6. Move File (with Duplicate Detection)
```
PUT /file-management.files/:fileId
```

**Request Body:**
```json
{
  "folderId": "TARGET_FOLDER_ID",
  "duplicateAction": "replace"
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| folderId | string | Yes | Target folder ID (null = root) |
| name | string | No | Rename file when moving |
| duplicateAction | string | No | "replace" / "keep_both" (only needed when duplicate exists) |

If `duplicateAction` is not provided and a duplicate exists, the server returns a `DUPLICATE_AT_DESTINATION` error with conflict info for the client to display a dialog.

---

#### 7. Delete File
```
DELETE /file-management.files/:fileId
```

Deletes file from both MinIO storage and MongoDB.

**Response:**
```json
{
  "success": true,
  "message": "File deleted successfully"
}
```

---

#### 8. Download File
```
GET /file-management.files/:fileId/download
```

**Response:** Binary file data

---

#### 9. Update File Content
```
POST /file-management.files/:fileId/update-content
```

Updates text file content (markdown, code files). Replaces content on MinIO and updates file size.

**Request Body:**
```json
{
  "content": "# Updated Markdown Content\n\nNew content here..."
}
```

**Response:**
```json
{
  "success": true,
  "message": "File content updated successfully",
  "file": {
    "_id": "FILE_ID",
    "file_size": 1024
  }
}
```

---

#### 10. Search Files
```
GET /file-management.files.search/:channelId
```

**Query Parameters:**
| Parameter | Type | Required | Default |
|-----------|------|----------|---------|
| q | string | Yes | - |
| limit | number | No | 20 |

**Response:**
```json
{
  "success": true,
  "files": [ /* File Object[] */ ],
  "query": "search term",
  "count": 5
}
```

---

#### 11. Filter Files (Advanced)
```
GET /file-management.files.filter/:channelId
```

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| folderId | string | Filter by folder |
| fileType | string | File category (see table below) |
| userId | string | Filter by uploader |
| dateRange | string | today/yesterday/thisweek/thismonth/lastmonth |
| sizeMin | number | Min file size (bytes) |
| sizeMax | number | Max file size (bytes) |
| search | string | Text search in filename |
| sortBy | string | name/size/type/created_at |
| sortOrder | string | asc/desc |
| count | number | Items per page |
| offset | number | Pagination offset |

**File Type Categories:**
| Value | Description | Extensions |
|-------|-------------|------------|
| `all` | All files | - |
| `image` | Images | jpg, jpeg, png, gif, bmp, webp, svg |
| `document` | Documents | pdf, doc, docx, txt, rtf, odt, xls, xlsx, ppt, pptx |
| `video` | Videos | mp4, avi, mov, wmv, flv, webm, mkv |
| `audio` | Audio | mp3, wav, flac, aac, ogg, m4a |
| `archive` | Archives | zip, rar, 7z, tar, gz |
| `code` | Code files | js, ts, html, css, py, java, cpp, c, go |

**Response:**
```json
{
  "success": true,
  "files": [ /* File Object[] */ ],
  "count": 20,
  "offset": 0,
  "total": 100,
  "filters": { /* Applied filters */ }
}
```

---

#### 12. Get Recent Files
```
GET /file-management.files.recent/:channelId
```

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| folderId | string | - | Filter by folder |
| hours | number | 24 | Time window (hours) |
| count | number | 20 | Max results |
| fileType | string | all | File type filter |

---

#### 13. Batch Embed Files
```
POST /file-management.files.batch-embed
```

Triggers AI embedding for files that haven't been embedded yet. Processes up to 100 files per batch.

**Request Body:**
```json
{
  "channelId": "CHANNEL_ID",
  "folderId": "FOLDER_ID"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Batch embedding completed: 45 processed, 2 failed",
  "processed": 45,
  "failed": 2
}
```

---

### Folder Operations

#### 14. Create Folder
```
POST /file-management.folders.create
```

**Request Body:**
```json
{
  "name": "Documents",
  "channelId": "CHANNEL_ID",
  "fatherId": "PARENT_FOLDER_ID"
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| name | string | Yes | Folder name |
| channelId | string | Yes | Channel ID |
| fatherId | string | No | Parent folder ID (omit/null = root) |

**Response:**
```json
{
  "success": true,
  "folder": { /* Folder Object */ }
}
```

---

#### 15. Get Folders in Channel
```
GET /file-management.folders.channel/:channelId
```

**Query Parameters:**
| Parameter | Type | Default |
|-----------|------|---------|
| fatherId | string | - |
| count | number | 50 |
| offset | number | 0 |

**Response:**
```json
{
  "success": true,
  "folders": [ /* Folder Object[] */ ],
  "count": 10,
  "offset": 0,
  "total": 50
}
```

---

#### 16. Get All Folders (Hierarchical)
```
GET /file-management.folders.all/:channelId
```

Gets all folders in the channel in a single request (useful for folder tree UI, move dialog).

**Response:**
```json
{
  "success": true,
  "folders": [ /* Folder Object[] - all folders */ ],
  "count": 50
}
```

---

#### 17. Get Folder by ID
```
GET /file-management.folders/:folderId
```

**Response:**
```json
{
  "success": true,
  "folder": { /* Folder Object */ }
}
```

---

#### 18. Get Folder Content
```
GET /file-management.folders/content/:fatherId
```

Gets folders and files inside a folder.

**Query Parameters:**
| Parameter | Type | Default |
|-----------|------|---------|
| count | number | 50 |
| offset | number | 0 |

**Response:**
```json
{
  "success": true,
  "folder": { /* Folder Object */ },
  "childFolders": [ /* Folder Object[] */ ],
  "files": [ /* File Object[] */ ],
  "count": 15
}
```

---

#### 19. Search Folders
```
GET /file-management.folders.search/:channelId
```

**Query Parameters:**
| Parameter | Type | Required | Default |
|-----------|------|----------|---------|
| q | string | Yes | - |
| limit | number | No | 20 |

**Response:**
```json
{
  "success": true,
  "folders": [ /* Folder Object[] */ ],
  "query": "search term",
  "count": 3
}
```

---

#### 20. Rename Folder
```
PUT /file-management.folders/:folderId/rename
```

**Request Body:**
```json
{
  "name": "New Folder Name"
}
```

**Response:**
```json
{
  "success": true,
  "folder": { /* Folder Object */ },
  "message": "Folder renamed successfully"
}
```

---

#### 21. Move Folder (with Duplicate Detection)
```
PUT /file-management.folders/:folderId
```

**Request Body:**
```json
{
  "fatherId": "NEW_PARENT_ID",
  "duplicateAction": "replace"
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| fatherId | string | Yes | New parent folder ID (null/empty = root) |
| name | string | No | Rename when moving |
| duplicateAction | string | No | "replace" / "keep_both" (only needed when duplicate exists) |

Automatically updates folder_path of all child folders when moving.

---

#### 22. Delete Folder
```
DELETE /file-management.folders/:folderId
```

**Query Parameters:**
| Parameter | Type | Default |
|-----------|------|---------|
| recursive | boolean | false |

**Response:**
```json
{
  "success": true,
  "message": "Folder and 5 subfolders deleted successfully",
  "deletedCount": 5
}
```

---

### Channel, Statistics & Admin

#### 23. Get Channel Root Content
```
GET /file-management.channels/:channelId/root
```

Gets all root folders and files of a channel.

**Query Parameters:**
| Parameter | Type | Default |
|-----------|------|---------|
| count | number | 50 |
| offset | number | 0 |

**Response:**
```json
{
  "success": true,
  "folders": [ /* Folder Object[] */ ],
  "files": [ /* File Object[] */ ],
  "count": {
    "folders": 5,
    "files": 20,
    "total": 25
  },
  "total": {
    "folders": 10,
    "files": 100,
    "total": 110
  }
}
```

---

#### 24. Get File Statistics
```
GET /file-management.stats/:channelId
```

**Query Parameters:**
| Parameter | Type | Default |
|-----------|------|---------|
| folderId | string | - |

**Response:**
```json
{
  "success": true,
  "channelId": "CHANNEL_ID",
  "stats": {
    "totalFiles": 150,
    "totalStorage": 524288000,
    "fileTypes": [
      { "_id": "pdf", "count": 45 },
      { "_id": "jpg", "count": 30 }
    ],
    "topUploaders": [
      { "_id": "USER_ID", "count": 50 }
    ],
    "dateDistribution": {
      "today": 5,
      "thisWeek": 25,
      "thisMonth": 100
    }
  }
}
```

---

#### 25. Get Uploader Statistics
```
GET /file-management.uploaders/:channelId
```

Gets upload statistics by user with details (file count, total size, last upload).

**Query Parameters:**
| Parameter | Type | Default |
|-----------|------|---------|
| limit | number | 10 |

**Response:**
```json
{
  "success": true,
  "channelId": "CHANNEL_ID",
  "uploaders": [
    {
      "_id": "USER_ID",
      "fileCount": 25,
      "totalSize": 524288000,
      "lastUpload": "2024-01-15T10:00:00Z",
      "user": {
        "_id": "USER_ID",
        "username": "john_doe",
        "name": "John Doe",
        "status": "online",
        "avatarETag": "etag"
      }
    }
  ],
  "count": 10
}
```

---

#### 26. Get File Types
```
GET /file-management.file-types
```

Gets the list of file type categories for filtering.

**Response:**
```json
{
  "success": true,
  "fileTypes": [
    { "value": "all", "label": "All Files", "icon": "file" },
    { "value": "image", "label": "Images", "icon": "image", "extensions": ["jpg", "png"] },
    { "value": "document", "label": "Documents", "icon": "doc", "extensions": ["pdf", "docx"] }
  ]
}
```

---

#### 27. Get Date Ranges
```
GET /file-management.date-ranges
```

Gets the list of date range options for filtering.

**Response:**
```json
{
  "success": true,
  "dateRanges": [
    { "value": "today", "label": "Today" },
    { "value": "yesterday", "label": "Yesterday" },
    { "value": "thisweek", "label": "This Week" },
    { "value": "thismonth", "label": "This Month" },
    { "value": "lastmonth", "label": "Last Month" }
  ]
}
```

---

#### 28. Get Users for Filtering
```
GET /file-management.users/:channelId
```

Gets the list of users who have uploaded files in the channel.

**Query Parameters:**
| Parameter | Type | Default |
|-----------|------|---------|
| limit | number | 50 |

**Response:**
```json
{
  "success": true,
  "users": [
    {
      "_id": "USER_ID",
      "username": "john_doe",
      "name": "John Doe",
      "status": "online",
      "avatarETag": "etag",
      "fileCount": 25
    }
  ],
  "count": 10
}
```

---

#### 29. Sync from MinIO
```
POST /file-management.sync.from-minio
```

Scans the MinIO bucket and synchronizes with MongoDB: creates records for files not in DB, deletes records for files no longer on MinIO, removes duplicate records.

**Query Parameters:**
| Parameter | Type | Required |
|-----------|------|----------|
| channelId | string | Yes |

**Response:**
```json
{
  "success": true,
  "message": "MinIO sync completed",
  "stats": {
    "minioFilesFound": 150,
    "dbRecordsFound": 145,
    "filesCreated": 5,
    "foldersCreated": 2,
    "filesDeleted": 0,
    "foldersDeleted": 0,
    "duplicatesRemoved": 3,
    "errors": 0
  }
}
```

---

#### 30. Reinitialize MinIO Client (Admin Only)
```
POST /file-management.minio.reinitialize
```

Requires `manage-settings` permission. Use after updating MinIO settings.

**Response:**
```json
{
  "success": true,
  "message": "MinIO client reinitialized successfully",
  "configured": true,
  "connected": true
}
```

---

## Error Codes

| Error Code | Description |
|------------|-------------|
| `error-unauthorized` | Invalid or missing token |
| `error-file-not-found` | File does not exist |
| `error-folder-not-found` | Folder does not exist |
| `error-forbidden` | No access permission |
| `error-upload-failed` | Upload failed |
| `error-invalid-params` | Invalid parameters |
| `error-quota-exceeded` | Storage limit exceeded |
| `error-rate-limited` | Too many requests |
| `DUPLICATE_FILE` | File with same name exists in the same folder (upload) |
| `DUPLICATE_AT_DESTINATION` | File/folder with same name at destination (move) |
| `error-init-failed` | Chunked upload initialization failed |
| `error-invalid-session` | Invalid chunked upload session |
| `error-upload-chunk-failed` | Chunk upload failed |
| `error-complete-failed` | Chunked upload completion failed |

**Error Response Format:**
```json
{
  "success": false,
  "error": "error-file-not-found"
}
```

---

## Common Use Cases

### 1. Browse Files & Folders
1. Get root content: `GET /file-management.channels/{channelId}/root`
2. Get folder content: `GET /file-management.folders/content/{folderId}`
3. Get all folders (for tree UI): `GET /file-management.folders.all/{channelId}`

### 2. Upload File (Direct)
`POST /file-management.files.upload` with multipart/form-data

### 3. Upload Large File (Chunked)
1. Init session: `POST /file-management.files.upload-chunked-init`
2. Upload each chunk: `POST /file-management.files.upload-chunk` (repeat)
3. Complete: `POST /file-management.files.upload-chunked-complete`
4. (Cancel if needed): `POST /file-management.files.upload-chunked-cancel`

### 4. Handle Duplicate Files
1. Upload file → Server returns `DUPLICATE_FILE` error with conflict info
2. Display dialog for user to choose: Replace / Keep Both / Cancel
3. Resend request with `duplicateAction` parameter

### 5. Search Files & Folders
- Files: `GET /file-management.files.search/{channelId}?q=query`
- Folders: `GET /file-management.folders.search/{channelId}?q=query`

### 6. Filter Files
`GET /file-management.files.filter/{channelId}?fileType=document&dateRange=thisweek`

### 7. File Management
- **Rename:** `PUT /file-management.files/{fileId}/rename`
- **Move:** `PUT /file-management.files/{fileId}` (update folderId, supports duplicate detection)
- **Delete:** `DELETE /file-management.files/{fileId}`
- **Download:** `GET /file-management.files/{fileId}/download`
- **Edit content:** `POST /file-management.files/{fileId}/update-content`

### 8. Sync MinIO Storage
`POST /file-management.sync.from-minio?channelId={channelId}`

---

## Quick Reference

### File Operations Summary
| Operation | Method | Endpoint |
|-----------|--------|----------|
| Upload (direct) | POST | `/file-management.files.upload` |
| Upload (chunked init) | POST | `/file-management.files.upload-chunked-init` |
| Upload (chunk) | POST | `/file-management.files.upload-chunk` |
| Upload (chunked complete) | POST | `/file-management.files.upload-chunked-complete` |
| Upload (chunked cancel) | POST | `/file-management.files.upload-chunked-cancel` |
| List | GET | `/file-management.files.channel/:channelId` |
| Get detail | GET | `/file-management.files/:fileId` |
| Rename | PUT | `/file-management.files/:fileId/rename` |
| Move | PUT | `/file-management.files/:fileId` |
| Delete | DELETE | `/file-management.files/:fileId` |
| Download | GET | `/file-management.files/:fileId/download` |
| Update content | POST | `/file-management.files/:fileId/update-content` |
| Search | GET | `/file-management.files.search/:channelId` |
| Filter | GET | `/file-management.files.filter/:channelId` |
| Recent | GET | `/file-management.files.recent/:channelId` |
| Batch embed | POST | `/file-management.files.batch-embed` |

### Folder Operations Summary
| Operation | Method | Endpoint |
|-----------|--------|----------|
| Create | POST | `/file-management.folders.create` |
| List | GET | `/file-management.folders.channel/:channelId` |
| Get all | GET | `/file-management.folders.all/:channelId` |
| Get by ID | GET | `/file-management.folders/:folderId` |
| Get content | GET | `/file-management.folders/content/:fatherId` |
| Search | GET | `/file-management.folders.search/:channelId` |
| Rename | PUT | `/file-management.folders/:folderId/rename` |
| Move | PUT | `/file-management.folders/:folderId` |
| Delete | DELETE | `/file-management.folders/:folderId` |

### Other Endpoints
| Operation | Method | Endpoint |
|-----------|--------|----------|
| Root content | GET | `/file-management.channels/:channelId/root` |
| Statistics | GET | `/file-management.stats/:channelId` |
| Uploader stats | GET | `/file-management.uploaders/:channelId` |
| File types | GET | `/file-management.file-types` |
| Date ranges | GET | `/file-management.date-ranges` |
| Users list | GET | `/file-management.users/:channelId` |
| Sync MinIO | POST | `/file-management.sync.from-minio` |
| Reinit MinIO | POST | `/file-management.minio.reinitialize` |

---

## Version

**Current Version:** v2.0.0 (2025-03-18)

**Changelog from v1.0.0:**
- Chunked upload for large files (>= `FileUpload_MaxDirectFileSize`, default 100MB)
- Duplicate file detection and conflict resolution (upload and move)
- File content editing (markdown, code files)
- Folder search endpoint
- Get all folders endpoint (hierarchical)
- Get folder by ID endpoint
- Uploader statistics endpoint
- MinIO sync endpoint
- MinIO reinitialize endpoint (admin)
- Webhook triggers for file/folder events
