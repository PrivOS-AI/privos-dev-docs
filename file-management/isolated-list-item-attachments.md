# Isolated-list item attachments — read/write authorization

List-item attachments are stored in a per-item folder at
`Uploads/list-items/{itemId}/`. The `privos_folders` record for that folder
carries an `itemId` metadata field; every file under it inherits the item's
authorization through the folder. There is no per-file ACL — access is derived
from the folder's `itemId` → the item → the item's list.

When the list is **not** isolated, item files behave like any other
channel-public upload (the usual room-membership / shared-folder gates apply).
When the list **is** isolated, each item is visible only to the room
owner/admin, the item creator, its assignees, and holders of a custom
permission listed in the item's `additionalReaders` / `additionalEditors`
(read cascades through the parent chain; see
[`ROOM_CUSTOM_PERMISSIONS`](../ROOM_CUSTOM_PERMISSIONS.md) and the isolated-list
item grant model). Attachment authorization mirrors that item-level decision.

## Two predicates, two intents

Both live in `apps/meteor/app/api/server/lib/list-item-folder-access.ts` and
delegate to the isolated-list item predicates in `isolated-list-item-filter.ts`.
Each short-circuits to `true` for a non-item folder, a deleted item, a missing
list, or a non-isolated list (write then falls back to the route's own
room-write gate).

| Predicate | Delegates to | Grants when (isolated list) |
|-----------|--------------|------------------------------|
| `canAccessItemFolder` / `canAccessFileByItemFolder` (**READ**) | `isItemVisibleInIsolatedList` | owner/admin, creator, assignee, `additionalReaders` **or** `additionalEditors`, or a readable ancestor |
| `canWriteItemFolder` / `canWriteFileByItemFolder` (**WRITE**) | `isItemWritableInIsolatedList` | owner/admin, creator, assignee, or `additionalEditors` — **`additionalReaders` never grants write**, and write does not cascade through ancestors |

The distinction matters: a user who can only *see* an item (a read grant, or a
sibling-item assignee) must not be able to upload into, rename, move, or delete
that item's attachments. Using the read predicate for a mutation would let a
read-only grantee change another user's isolated content, and a move could
relocate the file out to a channel-public folder.

## Route coverage

All routes are in `apps/meteor/app/api/server/v1/fileManagement.ts`. Read gates
were already present; the write gates close the mutation side.

**Read (visibility) — `canAccessItemFolder` / `canAccessFileByItemFolder`:**
`GET files/:fileId`, `/download`, `/content`, clone, folder listing, folder
info, search, recent files. The unauthenticated inline path refuses any file in
an isolated item folder outright (`isFileInIsolatedItemFolder`). Shared-surface
agent viewers use the role-independent `createSharedSurfaceItemFolderFilter`,
which drops isolated item files by list membership regardless of an elevated
room role.

**Write (mutation) — `canWriteItemFolder` / `canWriteFileByItemFolder`:**

| Route | Gate |
|-------|------|
| `POST` upload (into item folder) | `canWriteItemFolder(destFolder)` |
| `PUT files/:fileId` (rename/move) | `canWriteFileByItemFolder(source)` + `canWriteItemFolder(destFolder)` on move |
| `PUT files/:fileId/rename` | `canWriteFileByItemFolder(file)` |
| `DELETE files/:fileId` | `canWriteFileByItemFolder(file)` |
| `PUT folders/:folderId` (rename/move) | `canWriteItemFolder(folder)` |
| `PUT folders/:folderId/rename` | `canWriteItemFolder(folder)` |
| `DELETE folders/:folderId` | `canWriteItemFolder(folder)` |

All write gates run **after** the existing room-write / shared-folder gates, so
they only ever tighten access, never widen it.

## Known ceilings

- Both predicates fail **open** when the item or list is already deleted
  (`return true`) so cleanup paths (`deleteItemFolder`) and orphaned folders are
  not blocked. An orphaned isolated item file is therefore room-accessible until
  `deleteItemFolder` finishes purging it — by design, same as the pre-existing
  read gate.
- Item folders are flat (files only, no sub-folders), so the folder-level write
  gate protects the item folder itself; per-file placement inside it is covered
  by the upload gate.

Covered by `apps/meteor/app/api/server/lib/list-item-folder-access.spec.ts`.
