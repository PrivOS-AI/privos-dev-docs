# Stable File `_id` and Replace Semantics

Status: active — implemented 2026-05-13.

## Why

Links to file-management files use the Mongo `_id` of the file record
(`/api/v1/file-management.files/:fileId`). Anything that holds that id —
chat messages, AI chat citations, embedding rows, agent outputs, MCP
tool results — breaks if the server allocates a new `_id` for what is
semantically the same file.

Before this change, replacing a file (via UI choice "replace existing"
or via a sandbox agent regenerating an artifact) was implemented as
`deleteOne + insertOne`, which minted a new `_id` and silently broke
every existing link.

## What changed

Replace is now an in-place upsert keyed by `(channel_id, file_path)`.
The Mongo `_id` is invalidated only by an explicit delete.

Two server-side endpoints were updated in
`apps/meteor/app/api/server/v1/fileManagement.ts`:

1. `POST /api/v1/file-management.files.upload` — when
   `duplicateAction === 'replace'`, the existing record is reused.
   MinIO is overwritten at the same key. Embedding/parse state is
   cleared so the new content is re-indexed. A `file.updated` webhook
   is fired instead of `file.created`.
2. `POST /api/v1/file-management.files.upload-chunked-init` /
   `…upload-chunked-complete` — init no longer deletes; complete
   detects an existing record by `(channel_id, file_path)` and
   upserts. Same webhook switch.

The unique index `uniq_channel_filepath` on
`(channel_id, file_path)` guarantees the lookup matches at most one
row.

## Sandbox flow (recommended)

When a sandbox agent regenerates a file under `@Files/<...>/<name>`,
**do not** delete-then-upload. Use one of:

### Option A — let the duplicate-detection upsert handle it

```http
POST /api/v1/file-management.files.upload
multipart/form-data:
  files=<bytes>
  channelId=<room id>
  folderId=<containing folder _id, or omit for root>
  duplicateAction=replace
```

Server upserts on `(channel_id, folder_id, name)`. Existing `_id` is
preserved. Old links keep working.

### Option B — direct content update by id (preferred when the agent
already knows the file id)

```http
# 1. Resolve current id by path:
GET /api/v1/file-management.resolve-path/:channelId?path=InvoiceDemo/generated/2026-05-11/invoice-template-filled.docx
→ { "_id": "6a01a5b0635ad2b6c02bd63d", "type": "file", "name": "invoice-template-filled.docx" }

# 2. Overwrite content in place:
POST /api/v1/file-management.files/:fileId/update-binary-content
multipart/form-data:
  file=<bytes>
```

Both paths keep the file's `_id` stable.

## What still rotates the `_id`

Only explicit `DELETE /api/v1/file-management.files/:fileId` (or the
equivalent UI action). If a caller deletes a file and then uploads a
new file with the same name, that is a genuinely new file and gets
a new `_id` — which is the correct behavior.

## Caveats

- Path-derived ids are **not** used. `_id` is still a Mongo ObjectId.
  We considered using the MinIO key (`bucket + object key`) as `_id`
  but rejected it: MinIO has no separate file id, the key changes on
  rename/move, and keys contain characters that are awkward in URLs
  and Mongo `_id`s. Keeping the ObjectId stable across replace is
  sufficient.
- Do not pass folder `_id`s to `file-management.files/:fileId`.
  Folders and files live in different collections. When a folder id
  is passed to the files endpoint the server now returns:

  ```json
  {
    "success": false,
    "error": "File not found",
    "hint": "id matches a folder, not a file",
    "folder": { "_id": "...", "name": "...", "channel_id": "..." },
    "endpoint": "/api/v1/file-management.folders/<id>"
  }
  ```

  Use `resolve-path` first when in doubt, or follow the `endpoint`
  field if you really meant the folder.
