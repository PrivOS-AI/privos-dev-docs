# File Management - Current Features & Workflows

## Overview

File Management is a complete file and folder management system within channels, including:

- File storage on MinIO (S3-compatible storage)
- Metadata stored on MongoDB (separate database, independent from main Rocket.Chat DB)
- AI embedding support for smart search
- Hierarchical folder structure (unlimited depth)
- Chunked upload for large files (exceeding `FileUpload_MaxDirectFileSize` setting, default 100MB)
- Duplicate file detection and conflict resolution
- File viewing/editing directly in the app
- Webhook triggers for file/folder events
- MinIO sync to synchronize storage with database

---

## System Architecture

### Storage Layer
- **MinIO** (S3-compatible): Stores file content
- **MongoDB** (separate connection): Stores metadata (collections: `privos_files`, `privos_folders`, `privos_archive_files`)
- Presigned URLs for downloads (bypasses server proxy)
- Timestamp-prefixed filenames to ensure uniqueness

### Frontend
- React components in `/apps/meteor/client/views/room/contextualBar/FileManagement/`
- Client service: `FileManagementClient` class (`/apps/meteor/client/lib/fileManagementClient.ts`)
- React Query for data fetching with auto-refetch
- localStorage for user preferences (auto-parse per-channel, dotfiles visibility per-channel)

### Backend
- REST API v1 endpoints in `/apps/meteor/app/api/server/v1/fileManagement.ts`
- Database models: `FilesRaw`, `FoldersRaw`, `ArchiveFilesRaw`
- Webhook triggers for bot integrations

---

## File Upload Features

### 1. Direct Upload (Files < `FileUpload_MaxDirectFileSize`)
- Upload files to a channel or specific folder
- Supports all file types
- Upload progress tracking
- Auto Parse with AI on upload (can be toggled per-channel, saved in localStorage)

### 2. Chunked Upload (Files >= `FileUpload_MaxDirectFileSize`)
For large files that exceed the configurable threshold (server setting `FileUpload_MaxDirectFileSize`, default 100MB):
1. **Init session** - Server returns sessionId, chunkSizes, totalChunks
2. **Upload each chunk** - Send each part with checksum verification
3. **Complete** - Merge chunks and upload to MinIO
4. **Cancel** - Cancel session and cleanup temp files

### 3. Folder Upload
- Upload entire folders (preserving directory structure)
- Automatically creates corresponding folders on the server
- Preserves directory hierarchy

### 4. Drag-and-Drop Upload
- Drag and drop files/folders directly into the UI
- Supports both single and multiple files

### 5. Duplicate File Detection
When uploading a file with the same name in the same folder:
- Server detects the duplicate and returns conflict info
- Client displays a dialog for user to choose:
  - **Replace** - Delete the old file, upload the new one
  - **Keep Both** - Keep both files, auto-rename with suffix `_1`, `_2`...
  - **Cancel** - Cancel upload
- Backend is the authoritative source for duplicate checks (not client-side)

**Workflow:**
```
User uploads file → Server checks duplicate → Duplicate found?
  ├─ No → Upload normally
  └─ Yes → Return DUPLICATE_FILE error
          → Client shows DuplicateFileDialog
          → User chooses action (replace/keep_both/cancel)
          → Client resends request with duplicateAction parameter
```

---

## Folder Management Features

### 1. Folder Structure
- Hierarchical folder structure (parent-child)
- Unlimited folder depth
- Folders can be at root level or inside other folders

### 2. Folder Operations
- **Create** - Create new folder at any level
- **Rename** - Rename folder, automatically updates folder_path
- **Move** - Move folder to a different parent with duplicate detection
  - Supports duplicate detection similar to files (Replace/Keep Both/Cancel)
  - Automatically updates folder_path of all child folders
- **Delete** - Delete folder with options:
  - Delete only empty folder
  - **Recursive delete** - Delete folder and all subfolders/files inside

### 3. Folder Navigation
- **Breadcrumb navigation** with animation
- Browse folders by hierarchy
- Get root content (folders + files at the top level)
- Get folder content (child folders + files)
- **Get all folders** in a single request (for folder tree UI, move dialog)

### 4. Folder Search
- Search folders by name within a channel
- Case-insensitive search

---

## Dotfiles Visibility

### Hidden Files/Folders Toggle
Files and folders starting with `.` (dotfiles) are hidden by default in the file listing:
- **Toggle button** (eye/eye-off icon) in the toolbar to show/hide dotfiles
- **Default state**: hidden — dotfiles are filtered out
- **Applies to both** files AND folders (e.g., `.markdown`, `.config`, `.env`)
- **Preference persisted** per-channel per-user in localStorage (`filemanagement_showdotfiles_{roomId}`)
- Works alongside existing type/user/time/search filters (dotfile filter applied first)

**Note:** This is a client-side visibility filter only. Dotfiles remain accessible via API regardless of the toggle state. If a user navigates into a dotfile folder (e.g., `.markdown`), the folder contents are still visible — only the listing in the parent folder is affected.

---

## Search & Filter Features

### 1. Search Files
- Search by filename (case-insensitive)
- Search across the entire channel
- Configurable result limit

### 2. Search Folders
- Search folders by name
- Supports result limiting

### 3. Advanced Filtering
Filter files by multiple criteria simultaneously:

**By File Type:**
| Category | Extensions |
|----------|-----------|
| Images | jpg, jpeg, png, gif, bmp, webp, svg |
| Documents | pdf, doc, docx, txt, rtf, odt, xls, xlsx, ppt, pptx |
| Videos | mp4, avi, mov, wmv, flv, webm, mkv |
| Audio | mp3, wav, flac, aac, ogg, m4a |
| Archives | zip, rar, 7z, tar, gz |
| Code | js, ts, html, css, py, java, cpp, c, go, rust |

**By Date Range:**
- Today, Yesterday, This week, This month, Last month

**By User:**
- Filter by uploader (dropdown with avatar and file count)

**By Size:**
- Minimum / maximum file size

**By Text:**
- Search within filenames

**Sorting:**
- Sort by: Name, Size, Type, Date
- Order: Ascending, Descending

### 4. Recent Files
- Get recently uploaded files
- Filter by time window (hours)
- Filter by file type

---

## File Viewing & Editing Features

### 1. General File Viewer
- Display file metadata (size, type, date, uploader)
- File download
- File info sidebar

### 2. Image Gallery
- Lightbox preview for images
- Gallery mode when multiple images are present

### 3. Markdown Viewer & Editor
- **WYSIWYG view mode** for `.md`, `.markdown`, `.mdx` files
- **Code edit mode** with syntax highlighting
- Toast UI Editor integration
- **Mermaid diagram rendering** within markdown
- **Export** markdown to various formats (PDF, HTML, docx...)
- **Shared Canvas integration** - publish/share markdown content

### 4. Code File Editing
- Syntax highlighting for code files
- Edit directly in the app
- Save content via `update-content` API

### 5. File Attachments in Messages
- Generic file attachment display
- Image preview in messages
- Video player
- Audio player
- Upload progress indicator in messages
- Lazy loading for images

---

## AI Document Parsing (ocr-worker integration)

> Migrated from Flowise/PrivOS Connect to a dedicated **ocr-worker** REST service.
> Config resolves from a workspace-wide **Admin > Settings > Document Parser** group,
> with an optional per-room override (mirrors the PrivOS Sandbox "Use Global" pattern).

### 1. Config Resolution (global vs. room)
- **Global admin settings** (`Document_Parser` group): `DocumentParser_URL`,
  `DocumentParser_API_Key` (secret), `DocumentParser_Use_Env` (read
  `DOCUMENT_PARSER_URL` / `DOCUMENT_PARSER_API_KEY` env vars instead of DB).
- **Per-room override**: `rooms.customFields.documentParser = { useGlobal, url, apiKey }`.
- Effective config resolved by `getEffectiveDocumentParserConfig(roomId)`:
  - `DocumentParser_Use_Env` true → env (`source: 'env'`)
  - room `useGlobal === true`, or room has no url+key → admin global (`source: 'global'`)
  - otherwise → room's own url/key (`source: 'room'`)
  - `useGlobal:false` with no stored url/key still falls back to global.
- Legacy rooms with `{ url, apiKey }` and no `useGlobal` field keep their room
  override (backward compatible; no migration needed).
- API key never leaves the server: client only sees `{ useGlobal, hasApiKey }`.
- Endpoints:
  - `GET  /v1/rooms.isDocumentParserConfigured` — resolver-backed configured check
  - `POST /v1/rooms.testDocumentParserConnection` — room health-check (accepts `useGlobal`)
  - `POST /v1/rooms.saveDocumentParserConfig` — validate + save (accepts `useGlobal`)
  - `GET/POST /v1/document-parser.adminConfig` — read/save global admin settings
  - `POST /v1/document-parser.testAdminConnection` — admin health-check
- Room "Validate & Save" + a "Use global Document Parser config" toggle in
  Advanced settings; admin page has its own URL/key + Test Connection. Both support
  the `__use_existing__` sentinel to avoid resending the stored API key.

### 2. Auto Parse Toggle
- UI label "Auto Parse" (formerly "Auto-Embed")
- **Toggle is gated on effective doc-parser config**: greyed out + info tooltip when
  neither the room nor the global admin endpoint is configured (checked via
  `rooms.isDocumentParserConfigured`)
- Tooltip text: "Document Parser not configured. Go to Room Settings → Advanced
  to set up the ocr-worker endpoint and enable Auto Parse."
- When enabled, every uploaded / replaced / content-updated file is auto-parsed
- Toggle per-user, per-room (localStorage key `filemanagement_autoembed_{roomId}`)

### 3. Parse Lifecycle
The ocr-worker REST contract:
- `POST /api/v1/jobs/process` (multipart) — upload file bytes → returns `{ job_id }`
- `GET /api/v1/jobs/status/{job_id}` — poll status (`waiting | active | completed | failed`)
- `POST /api/v1/jobs/delete-file` — remove parsed `.markdown/` artefact when source file is deleted

privos-hub helper functions in `fileManagement.ts`:
- `triggerFileParse(channelId, fileId, presignedUrl, filename, filePath)` — submits
  parse job and marks file `parse_status: 'pending'`
- `triggerFileParseDelete(channelId, fileId, filename, filePath)` — fire-and-forget
  cleanup of `.markdown/` on deletion

These are called automatically on:
- File upload (direct + chunked completion)
- File content update (text + binary)
- Duplicate-replace upload
- File delete (single + folder recursive)

### 4. Parse Status Field
Each file document has:
- `parse_status?: 'pending' | 'complete' | 'error' | 'timeout'`
- `parse_submitted_at?: Date` — used to detect 30-min timeout
- `embedding_task_id?: string` — BullMQ job id from ocr-worker
- `is_embedded?: boolean` — true when parsing completed

### 5. Background Parse-Status Poller
`server/services/parse-status-poller.ts` runs on a 5 s interval. For every file
with `parse_status === 'pending'` it calls the ocr-worker status endpoint and
updates `parse_status` to `complete` / `error`, or to `timeout` when
`parse_submitted_at` is older than 30 min.

### 6. UI Status Badge (per file)
A small badge is rendered at the bottom-right of every file icon:
- **Pending**: animated blue spinner
- **Timeout / Error**: red spinner — clickable; calls retry-parse endpoint
- **Complete** (or legacy `is_embedded === true`): blue tick
- Files inside the `.markdown` root folder hide the badge (excluded from parsing)

### 7. Retry From Menu
Both the right-click context menu and the `…` dropdown expose a "Manual Parse"
item that calls `POST /file-management.files/:fileId/retry-parse`. The item is
**always enabled**, even while `parse_status === 'pending'`, so users can
re-submit a stuck job.

### 8. Batch Parse
- `POST /file-management.files.batch-embed` — submits parse jobs for every
  un-parsed file in the room
- Excludes files inside the root-level `.markdown` folder (which contains the
  parsed output and must never be re-parsed)
- Fire-and-forget: returns immediately with `{ submitted: N }`; client polls
  via `check-parse-status`

### 9. Status Polling Endpoint
- `POST /file-management.files.check-parse-status` — body `{ channelId }`,
  returns the current file list. The Files tab calls it every 3 s while any
  file is `pending`, so badges update without a full refetch.

### 10. `.markdown` Folder Convention
The ocr-worker writes parsed artefacts under
`{room_id}/.markdown/{relative_path}.md` in the same MinIO bucket that holds
the source files. Path resolution priority:
1. `filename` already contains a `/` (archive flow) → use as-is
2. `file_path` (MinIO object key) — strip `{room_id}/` prefix
3. `download_link` URL extraction (legacy)
4. Plain `filename`

This preserves subfolder structure: `ROOM123/test/eVND.pdf`
→ `.markdown/test/eVND.md`.

**Workflow:**
```
Upload file
  → privos-hub stores bytes in MinIO at {room_id}/{folder}/{name}
  → triggerFileParse(): POST multipart to ocr-worker /jobs/process
  → privos-hub marks parse_status='pending' + parse_submitted_at=now
  → ocr-worker queues BullMQ job, returns {job_id}
  → Worker downloads / extracts → writes .markdown/{path}.md to MinIO
  → parse-status-poller picks up 'completed' → updates is_embedded=true,
                                                 parse_status='complete'

Delete file
  → triggerFileParseDelete(): POST to ocr-worker /jobs/delete-file
  → privos-hub deletes source from MinIO and MongoDB immediately
  → ocr-worker enqueues delete job (no scheduling delay)
  → Worker removes {room_id}/.markdown/{relative}.md
```

---

## MinIO Sync Features

### 1. Sync from MinIO to Database
Synchronize MinIO bucket with MongoDB:
- **Scan MinIO** → Find files without DB records → Create records
- **Scan DB** → Find records without files on MinIO → Delete records
- **Cleanup duplicates** → Find and remove duplicate records (keep the newest)
- Returns detailed statistics (filesCreated, foldersCreated, filesDeleted, foldersDeleted, duplicatesRemoved)

### 2. MinIO Reinitialize (Admin)
- Reinitialize MinIO client after changing settings
- Requires `manage-settings` permission
- Checks connection status after reinit

**Workflow:**
```
User clicks "Sync" → POST /sync.from-minio
  → Step 0: Clean up duplicate records in DB
  → Step 1: Scan MinIO bucket for the channel
  → Step 2: Create folder records for missing folders
  → Step 3: Create file records for missing files
  → Step 4: Delete records for files/folders no longer on MinIO
  → Return sync statistics
```

---

## Statistics Features

### 1. File Statistics
- **Total files** - Total number of files in channel/folder
- **Total storage** - Total storage used (bytes)
- **File type distribution** - Breakdown by file type
- **Top uploaders** - Users who uploaded the most
- **Date distribution** - Breakdown by time period (today, this week, this month)

### 2. Uploader Statistics
- List of users who uploaded with details:
  - File count
  - Total size
  - Last upload time
  - User profile info (username, name, avatar, status)

---

## Webhook Triggers

File Management fires webhook events on changes:

| Event | Trigger |
|-------|---------|
| `file.created` | File uploaded successfully |
| `file.updated` | File renamed or moved |
| `file.deleted` | File deleted |
| `folder.created` | New folder created |
| `folder.renamed` | Folder renamed |
| `folder.deleted` | Folder deleted |

Webhook events are fired in a "fire-and-forget" manner (does not block the main operation).

---

## Duplicate Handling on Move

When moving a file or folder, the server checks for duplicates at the destination:

**Move File:**
```
User moves file → Server checks duplicate at destination
  ├─ No duplicate → Move normally (copy MinIO + delete old + update DB)
  └─ Duplicate found → Return DUPLICATE_AT_DESTINATION error
         → Client shows DuplicateMoveDialog
         → User chooses action:
           ├─ Replace → Delete existing file at destination → Move file
           ├─ Keep Both → Rename with suffix → Move file
           └─ Cancel → Cancel move
```

**Move Folder:**
```
User moves folder → Server checks duplicate at destination
  ├─ No duplicate → Move (update father + folder_path + child paths)
  └─ Duplicate found → Return DUPLICATE_AT_DESTINATION error
         → User chooses action (same as above)
```

---

## Security & Access Control

### 1. Authentication
- All endpoints require authentication
- Headers: `X-Auth-Token`, `X-User-Id`

### 2. Authorization
- Channel-level access control
- Users can only access files in channels they belong to
- Folder operations validate channel ownership
- Chunked upload verifies session ownership
- MinIO reinitialize requires `manage-settings` permission

### 3. File Access
- Presigned URLs expire after a short period
- No direct access to MinIO credentials
- Filename sanitization (removes special characters)

---

## UI Components

### Main Components
| Component | Description |
|-----------|-------------|
| `FileManagement.tsx` | Main container - folder navigation, filters, upload controls, Auto Parse toggle, dotfiles toggle |
| `FileManagementList.tsx` | List rendering with loading states |
| `FileFolderGrid.tsx` | Grid display of folders and files |
| `FileListItem.tsx` | Individual file row with actions menu |
| `FileBreadcrumbs.tsx` | Folder path navigation with animation |
| `FileFilters.tsx` | Advanced filtering UI (type, user, date dropdown) |

### Modal/Dialog Components
| Component | Description |
|-----------|-------------|
| `UploadModal.tsx` | File upload with drag-drop, chunked upload, folder upload |
| `NewFolderModal.tsx` | Create new folder |
| `RenameModal.tsx` | Rename file/folder |
| `MoveModal.tsx` | Move file/folder with folder tree navigation |
| `DuplicateFileDialog.tsx` | Conflict resolution UI for upload (Replace/Keep Both/Cancel) |
| `DuplicateMoveDialog.tsx` | Conflict resolution UI for move operations |
| `SyncProgressDialog.tsx` | MinIO sync progress feedback |

### File Viewing Components
| Component | Description |
|-----------|-------------|
| `FileMarkdownViewer.tsx` | Markdown WYSIWYG + code edit, Mermaid, export |
| `FileInfoSidebar.tsx` | File metadata display |
| `FileViewerWithData.tsx` | Data wrapper for file viewer |

---

## Pagination & Performance

### 1. Pagination
- All list endpoints support pagination
- Parameters: `count` (items per page), `offset` (skip items)
- Default: 50 items per page, Max: 100
- Total count is returned for implementing pagination UI

### 2. Performance
- Presigned URLs for downloads (reduces server load)
- Efficient queries with MongoDB indexes:
  - Files: `folder_id`, `channel_id`, `name`, `created_at`, `is_embedded`, compound `(channel_id, folder_id)`
  - Folders: `father`, `name`, `channel_id`, `folder_path`, compound `(channel_id, father)`
- React Query with caching and auto-refetch
- Lazy loading with pagination
- Get all folders in a single API call (instead of multiple calls)

---

## Main Workflows

### Workflow 1: Browse & Navigate
```
1. Open FileManagement panel (contextual bar)
2. Load root content → Display folders + files
3. Click folder → Load folder content → Display breadcrumbs
4. Click breadcrumb → Navigate to the corresponding folder
5. Apply filters (type, user, date) → Refresh list
```

### Workflow 2: Upload File
```
1. Click "Upload" button or drag-drop files
2. Select files (or folder) → UploadModal appears
3. Toggle Auto Parse on/off
4. Click Upload → Progress tracking
5. If duplicate → DuplicateFileDialog → User chooses action
6. Upload complete → Refresh file list
7. If file >= `FileUpload_MaxDirectFileSize` (default 100MB) → Automatically switches to chunked upload
```

### Workflow 3: Move File/Folder
```
1. Click action menu on file/folder → Select "Move"
2. MoveModal displays folder tree (fetched from getAllFolders API)
3. Select destination folder
4. Click Move → Server checks for duplicate
5. If duplicate → DuplicateMoveDialog → User chooses action
6. Move complete → Refresh
```

### Workflow 4: Edit Markdown/Code File
```
1. Click markdown/code file → FileMarkdownViewer opens
2. Choose mode: WYSIWYG or Code Edit
3. Edit content
4. Save → POST /update-content → Saved to MinIO
5. (Optional) Export to PDF/HTML/docx
6. (Optional) Share via Shared Canvas
```

### Workflow 5: Sync MinIO
```
1. Click "Sync" button
2. SyncProgressDialog appears
3. POST /sync.from-minio → Server scans MinIO + DB
4. Server cleans up duplicates, creates/deletes records
5. Display sync stats (created, deleted, errors)
6. Refresh file list
```

### Workflow 6: Batch Document Parsing
```
1. Toggle "Auto Parse" ON (or click batch action)
2. POST /file-management.files.batch-embed
   → Server finds files where parse_status is null/error/timeout
     (excluding the .markdown folder)
   → For each: submitParseJob() POSTs the file bytes to ocr-worker
   → Marks parse_status='pending', stores embedding_task_id
3. Endpoint returns { submitted: N } immediately (fire-and-forget)
4. Client polls POST /file-management.files.check-parse-status every 3s
5. Badges flip from blue spinner → tick / red as ocr-worker completes
```

### Workflow 7: Configure Document Parser
```
1. Open Room Settings → Advanced → Document Parser
2. Enter ocr-worker URL (https://...) and API key
3. Click "Validate & Save"
   → POST /v1/rooms.saveDocumentParserConfig
   → Server validates URL, calls /health on ocr-worker, saves to customFields
4. Auto Parse toggle in Files tab becomes enabled
```

---

## Database Schema

### Files Collection (`privos_files`)
```typescript
{
  _id: ObjectId;
  name: string;                      // Original filename
  folder_id?: string | null;         // Parent folder (null = root)
  channel_id: string;                // Channel ID
  file_path?: string;                // MinIO storage path
  file_size?: number;                // Bytes
  file_type?: string;                // MIME type or extension
  user_id?: string;                  // Uploader
  is_embedded?: boolean;             // True once ocr-worker reports parse complete
  embedding_task_id?: string;        // BullMQ job id returned by ocr-worker
  parse_status?:                     // Current parse lifecycle state
    | 'pending'
    | 'complete'
    | 'error'
    | 'timeout';
  parse_submitted_at?: Date;         // When parse job was submitted (used for >30 min timeout)
  archive_id?: string;               // Archive reference
  created_at: Date;
  updated_at: Date;
}
```

### Folders Collection (`privos_folders`)
```typescript
{
  _id: ObjectId;
  name: string;                      // Folder name
  father?: string | null;            // Parent folder (null = root)
  channel_id: string;                // Channel ID
  folder_path?: string;              // Display path
  user_id?: string;                  // Creator
  created_at: Date;
  updated_at: Date;
}
```

---

## User Preferences (localStorage)

Per-channel, per-user preferences stored in localStorage:

| Key Pattern | Default | Description |
|-------------|---------|-------------|
| `filemanagement_autoembed_{roomId}` | Server setting | Auto Parse toggle state (formerly Auto-Embed) |
| `filemanagement_showdotfiles_{roomId}` | `false` | Dotfiles visibility toggle (hidden by default) |

These are personal preferences — not shared across users or synced to the server.

---

## Version

**Current Version:** v2.2.0 (2026-05-04)

**Changelog from v2.1.0:**
- Migrated document parsing from Flowise/PrivOS Connect to dedicated **ocr-worker** REST service
- Added per-room Document Parser config (`customFields.documentParser`) and Advanced settings UI
- New `parse_status` / `parse_submitted_at` fields on file documents
- New endpoints: `rooms.saveDocumentParserConfig`, `rooms.testDocumentParserConnection`, `file-management.files.check-parse-status`, `file-management.files/:fileId/retry-parse`
- `batch-embed` now fire-and-forget; excludes the `.markdown` folder
- Background `parse-status-poller` (5 s) plus client `check-parse-status` polling (3 s while pending)
- Parse-status badge on file icons (spinner / tick / red retry)
- Retry-parse always available from context menu / `…` dropdown, even while pending
- Auto Parse toggle gated on whether the room has a configured doc-parser endpoint
- ocr-worker delete-file path now removes the corresponding `.markdown/` artefact (no 30 s delay)
