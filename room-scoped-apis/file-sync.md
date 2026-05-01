# File Sync API

## Overview

Synchronize MinIO state → PrivOS Hub file management database (`privos_files`, `privos_folders`).

When an external service (AI Service, MCP Tools, etc.) writes files **directly to MinIO**, the PrivOS Hub database does not automatically know about them. This API allows the external service to send path hints so PrivOS Hub can sync the database to match MinIO's actual state.

**The server always reads MinIO as the source of truth — the payload is just a hint about which areas changed.**

**Endpoint:** `POST /api/v1/internal/rooms/:roomId/files/sync`

## Authentication

| Header | Description |
|--------|-------------|
| `x-api-key` | Internal API key |
| `Authorization` | `Bearer <JWT_TOKEN>` — Room-specific JWT token |

---

## Request

```http
POST /api/v1/internal/rooms/:roomId/files/sync
Content-Type: application/json
```

### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room/channel ID that owns the files |

### Body

```json
{
  "paths": [
    "Agent Artifacts/session1/report.md",
    "Agent Artifacts/session1/output/",
    "Agent Artifacts/session1/Dockerfile"
  ]
}
```

### Body Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `paths` | string[] | Yes | Non-empty list of paths relative to `{roomId}/` on MinIO |

### Path Rules

All paths are **relative to `{roomId}/`**. Do NOT include the roomId prefix.

```
MinIO full path:  abc123/Agent Artifacts/session1/report.md
                  ^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                  roomId  path (this is what you send)
```

### File vs Folder Detection

| Rule | Example | Result |
|------|---------|--------|
| Trailing `/` | `"Agent Artifacts/session1/"` | Folder |
| No trailing `/` | `"Agent Artifacts/session1/report.md"` | File |
| No trailing `/` (no extension) | `"Agent Artifacts/session1/Dockerfile"` | File |

**Rule: trailing `/` = folder, everything else = file.**

---

## How It Works

### File path → `checkAndSyncFile()`

1. Stats the object on MinIO (`getFileMetadata`)
2. **Object exists on MinIO:**
   - DB record exists, size unchanged → no-op
   - DB record exists, size changed → update `file_size` in DB
   - DB record missing → create DB record (self-healing)
3. **Object missing on MinIO:**
   - DB record exists → delete it
   - DB record missing → no-op

### Folder path → `syncFolder()`

1. Lists MinIO objects under the folder prefix (flat, non-recursive)
2. Compares with DB records for that folder
3. Inserts records for files found on MinIO but missing in DB
4. Updates `file_size` for files where size has changed
5. Deletes records for files in DB but no longer on MinIO
6. Auto-creates the folder hierarchy in DB if it doesn't exist

---

## Response

### Success

```json
{
  "success": true,
  "status": "success",
  "created": 2,
  "updated": 1,
  "deleted": 1,
  "errors": []
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Always `"success"` |
| `created` | number | Number of DB records created |
| `updated` | number | Number of DB records updated (size changed) |
| `deleted` | number | Number of DB records deleted |
| `errors` | string[] | Per-item errors. Empty if all succeeded. |

The sync is **partial-success** — if one item fails, others still proceed. Failures are reported in `errors[]`.

### Error Response

```json
{
  "success": false,
  "error": "Missing required field: paths (non-empty array)",
  "errorType": "error-invalid-params"
}
```

---

## Deduplication

If a folder path and file paths under that folder appear in the same batch, the file paths are **skipped** — the folder sync covers them.

```json
{
  "paths": [
    "Agent Artifacts/session1/output/",
    "Agent Artifacts/session1/output/result.md",
    "Agent Artifacts/session1/output/data.csv"
  ]
}
```

The two file paths are redundant — `syncFolder("Agent Artifacts/session1/output/")` already reconciles all files in that folder.

**Recommended:** when an entire folder changes, just send the folder path.

---

## Examples

### Example 1: AI wrote a new file

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/sync" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "paths": ["Agent Artifacts/session1/report.md"]
  }'
```

### Example 2: AI updated a file in place

Same payload as created — server stats MinIO and upserts the DB record automatically.

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/sync" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "paths": ["Agent Artifacts/session1/report.md"]
  }'
```

### Example 3: AI deleted a file

Send the file path. Server stats MinIO → file missing → removes DB record.

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/sync" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "paths": ["Agent Artifacts/session1/old-report.md"]
  }'
```

### Example 4: AI wrote multiple files to a folder

Send the folder path — server reconciles the entire folder with MinIO.

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/sync" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "paths": ["Agent Artifacts/session1/output/"]
  }'
```

### Example 5: AI made changes across multiple folders

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/sync" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "paths": [
      "Agent Artifacts/session1/output/",
      "Agent Artifacts/session1/archive/",
      "Documents/summary.md"
    ]
  }'
```

### Example 6: Files without extension

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/files/sync" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "paths": [
      "Agent Artifacts/session1/Dockerfile",
      "Agent Artifacts/session1/Makefile"
    ]
  }'
```

---

## Self-Healing Behavior

| Situation | Behavior |
|-----------|----------|
| File sent, parent folder doesn't exist in DB | Auto-creates folder hierarchy |
| File exists on MinIO, no DB record | Creates the record |
| File exists on MinIO, DB record exists, size unchanged | No-op |
| File exists on MinIO, DB record exists, size changed | Updates `file_size` |
| File missing on MinIO, DB record exists | Deletes the record |
| File missing on MinIO, no DB record | No-op |
| Folder sent, MinIO prefix empty | Deletes any orphan DB records in that folder |

---

## Best Practices

1. **Call sync after writing to MinIO** — not before. The server reads MinIO to determine the current state.

2. **Use trailing `/` for folders.** `"mydir/"` is always a folder. `"mydir"` without trailing slash is treated as a file regardless of extension.

3. **Send folder path when many files changed** — more efficient than listing every file individually.

4. **Batch all paths in a single request** when possible. One request with 10 paths is better than 10 requests.

5. **No need to distinguish created / updated / deleted** — the server derives the correct action from MinIO state automatically.

---

## MinIO Path Structure

```
bucket/
├── {roomId}/                          ← room root
│   ├── file-at-root.md
│   ├── Documents/
│   │   └── report.pdf
│   └── Agent Artifacts/               ← AI session artifacts
│       ├── {sessionId1}/
│       │   ├── canvas.md
│       │   ├── analysis.html
│       │   └── data/
│       │       └── output.csv
│       └── {sessionId2}/
│           └── summary.md
```

All paths in the request are relative to `{roomId}/`. To sync `canvas.md` above:

```json
{ "paths": ["Agent Artifacts/{sessionId1}/canvas.md"] }
```

To sync the entire `data/` folder:

```json
{ "paths": ["Agent Artifacts/{sessionId1}/data/"] }
```
