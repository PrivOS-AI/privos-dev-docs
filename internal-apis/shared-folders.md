# Shared Folders API

## Overview

Cross-room folder sharing. Links a source folder in one channel to a target channel with `readonly` or `writable` permission. Users browsing the target channel's File Management see shared folders at root level and can navigate into them.

## Authentication

All endpoints accept:
- **Session auth:** `X-Auth-Token` + `X-User-Id` headers
- **Bot bearer token:** `Authorization: Bearer <bot_token>` — bot must be a room member

## Database

**Collection:** `privos_shared_folders` (file management DB)

| Field | Type | Description |
|-------|------|-------------|
| `_id` | ObjectId | Auto-generated |
| `source_folder_id` | string | Folder being shared |
| `source_channel_id` | string | Channel that owns the folder |
| `target_channel_id` | string | Channel receiving the share |
| `permission` | string | `readonly` or `writable` |
| `shared_by` | string | User ID who created the share |
| `created_at` | Date | Creation timestamp |
| `updated_at` | Date | Last update timestamp |

**Unique constraint:** `(source_folder_id, target_channel_id)` — a folder can only be shared once per target channel.

**Indexes:** `target_channel_id`, `source_folder_id`, `source_channel_id`

## Authorization

- **Create:** Caller must be member of **both** channels. Must be **owner** or **moderator** of the target channel (global admin bypass).
- **Update permission:** Caller must be member of both channels (admin bypass).
- **List/Dismiss:** Caller must be a member of the target channel.
- **Read** (browsing files/folders): Allowed for any user with shared folder access, even if not a member of the source channel. Server resolves the correct source channel from the folder ID.
- **Write** (upload, create, rename, move, delete):
  - Source room members: full access regardless of share permission.
  - Non-members with `writable` share: allowed.
  - Non-members with `readonly` share: **blocked** server-side.
- **Cascade delete:** When a source folder is deleted, all share records referencing it are removed.
- **Nested sharing prevention:** Cannot share a folder that is inside an already-shared folder.

---

## Endpoints

### Create Shared Folder

Share a folder from one channel to another.

```
POST /api/v1/file-management.shared-folders.create
```

**Auth:** Required (session or bot token)

**Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sourceFolderId` | string | yes | ID of the folder to share |
| `sourceChannelId` | string | yes | Channel that owns the folder |
| `targetChannelId` | string | yes | Channel to share into |
| `permission` | string | yes | `readonly` or `writable` |

**Response (200):**

```json
{
  "success": true,
  "share": {
    "_id": "...",
    "source_folder_id": "...",
    "source_channel_id": "...",
    "target_channel_id": "...",
    "permission": "readonly",
    "shared_by": "...",
    "created_at": "...",
    "updated_at": "..."
  }
}
```

**Errors:**
- `Cannot share a folder to its own room` — source and target are the same channel
- `Not authorized to access source/target channel` — caller not a member
- `Only room owners or moderators can add shared folders` — insufficient role on target channel
- `Source folder not found` — invalid folder ID
- `Source folder does not belong to the specified channel`
- `Cannot share a folder that is inside an already-shared folder` — nested sharing blocked
- `This folder is already shared to the target room` — duplicate share

---

### List Shared Folders

Get all shared folders for a target channel, enriched with folder name and source room name.

```
GET /api/v1/file-management.shared-folders.channel/:channelId
```

**Auth:** Required (session or bot token)

**Response (200):**

```json
{
  "success": true,
  "shares": [
    {
      "_id": "...",
      "source_folder_id": "...",
      "source_channel_id": "...",
      "source_folder_name": "Design Assets",
      "source_channel_name": "design-team",
      "target_channel_id": "...",
      "permission": "readonly",
      "shared_by": "...",
      "can_edit_permission": true,
      "created_at": "...",
      "updated_at": "..."
    }
  ]
}
```

**Notes:**
- `can_edit_permission` is `true` when caller is a member of the source channel
- Stale shares (source folder deleted) are auto-cleaned during listing

---

### Update Permission

Toggle between `readonly` and `writable`.

```
PUT /api/v1/file-management.shared-folders/:shareId
```

**Auth:** Required (session or bot token, dual-room member or admin)

**Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `permission` | string | yes | `readonly` or `writable` |

**Response (200):**

```json
{
  "success": true,
  "share": { "...updated share record..." }
}
```

**Errors:**
- `Share not found`
- `Must be a member of both rooms to change permission`

---

### Dismiss Shared Folder

Remove a shared folder link from the target channel. Does not delete the source folder.

```
DELETE /api/v1/file-management.shared-folders/:shareId
```

**Auth:** Required (session or bot token, target channel member)

**Response (200):**

```json
{
  "success": true,
  "message": "Shared folder dismissed"
}
```

**Errors:**
- `Share not found`
- `Not authorized to dismiss this shared folder`
