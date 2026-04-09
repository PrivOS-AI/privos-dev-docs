# Privos MCP Tools — Folders

## `privos.folders.getByChannel`

Get folders in a channel, optionally filtered by parent folder.

| | |
|---|---|
| **Scope** | `files:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `channelId` | string | Yes | Channel/room ID |
| `parentId` | string | No | Filter by parent folder ID |
| `skip` | number | No | Pagination offset (default: 0) |
| `limit` | number | No | Max results (default: 50) |

### Response

```json
[
  {
    "_id": "folder_001",
    "name": "Documents",
    "channelId": "room_xyz789",
    "parentId": null,
    "createdBy": "user_123",
    "createdAt": "2026-03-25T10:00:00Z"
  }
]
```

---

## `privos.folders.get`

Get folder details by ID.

| | |
|---|---|
| **Scope** | `files:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `folderId` | string | Yes | Folder ID |

### Response

```json
{
  "_id": "folder_001",
  "name": "Documents",
  "channelId": "room_xyz789",
  "parentId": null,
  "createdBy": "user_123",
  "createdAt": "2026-03-25T10:00:00Z"
}
```

---

## `privos.folders.getContent`

Get all files and subfolders inside a folder.

| | |
|---|---|
| **Scope** | `files:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `folderId` | string | Yes | Folder ID |

### Response

```json
{
  "files": [
    { "_id": "file_001", "name": "document.pdf", "type": "application/pdf" }
  ],
  "folders": [
    { "_id": "folder_002", "name": "Subfolder" }
  ]
}
```

---

## `privos.folders.getRootContent`

Get root-level files and folders in a channel (items without a parent folder).

| | |
|---|---|
| **Scope** | `files:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `channelId` | string | Yes | Channel/room ID |

### Response

```json
{
  "files": [
    { "_id": "file_003", "name": "readme.txt" }
  ],
  "folders": [
    { "_id": "folder_001", "name": "Documents" },
    { "_id": "folder_004", "name": "Images" }
  ]
}
```

---

## `privos.folders.create`

Create a new folder.

| | |
|---|---|
| **Scope** | `files:write` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `channelId` | string | Yes | Channel/room ID |
| `name` | string | Yes | Folder name |
| `parentId` | string | No | Parent folder ID (null for root) |

### Response

```json
{
  "_id": "folder_005",
  "name": "New Folder",
  "channelId": "room_xyz789",
  "parentId": null,
  "createdBy": "user_123",
  "createdAt": "2026-03-25T10:00:00Z"
}
```

### Example

```typescript
// Create root folder
await app.callServerTool({
  name: 'privos.folders.create',
  arguments: { channelId: 'room_xyz789', name: 'Reports' }
});

// Create subfolder
await app.callServerTool({
  name: 'privos.folders.create',
  arguments: { channelId: 'room_xyz789', name: 'Q1 2026', parentId: 'folder_005' }
});
```

---

## `privos.folders.update`

Rename or move a folder.

| | |
|---|---|
| **Scope** | `files:write` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `folderId` | string | Yes | Folder ID |
| `name` | string | No | New folder name |
| `parentId` | string | No | Move to different parent folder |

### Response

Returns the updated folder object.

---

## `privos.folders.delete`

Delete a folder, optionally with all contents.

| | |
|---|---|
| **Scope** | `files:write` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `folderId` | string | Yes | Folder ID |
| `recursive` | boolean | No | Delete all contents recursively (default: false) |

### Response

```json
{ "deleted": true }
```

---

## `privos.folders.search`

Search folders by name in a channel.

| | |
|---|---|
| **Scope** | `files:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `channelId` | string | Yes | Channel/room ID |
| `query` | string | Yes | Search query |
| `limit` | number | No | Max results (default: 20) |

### Response

```json
[
  {
    "_id": "folder_001",
    "name": "Documents",
    "channelId": "room_xyz789"
  }
]
```
