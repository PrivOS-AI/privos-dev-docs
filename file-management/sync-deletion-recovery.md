# Sync Deletion Recovery (privos-sandbox recyclebin)

This document covers a class of bug where files deleted via the Hub UI
reappeared after the next AI chat attempt, and the recyclebin mechanism in
`privos-sandbox` that addresses it. The full design and implementation
details live in
[`privos-sandbox/docs/features/recyclebin.md`](../../../privos-sandbox/docs/features/recyclebin.md);
this page is the Hub-side summary.

## The bug

User-reported symptom: deleting files in the file-management panel succeeded,
but sending a message in AI chat made the files reappear.

Root cause was an asymmetric source-of-truth policy between the sandbox's
pull and push:

```
1. user deletes file in Hub UI
   → /api/v1/file-management.files/:fileId DELETE
   → MinIO object removed
   → privos_files row removed
2. user sends AI chat message
   → sandbox pull runs
     - sees remote object missing
     - keeps the local copy (sandbox's "preserve agent edits" policy)
3. agent finishes
   → sandbox push runs
     - sees a local-only file
     - uploads as reason: 'new'
4. file is back in MinIO
   → next file-management sync (`apps/meteor/server/lib/fileManagementSync.ts`)
     re-inserts the privos_files row
   → file reappears in the UI
```

The fix is in privos-sandbox: a per-workspace **recyclebin** archives the
local copy at the moment pull would have kept it, and the planner is now
allowed to prune. Push no longer sees the file, so it doesn't re-upload.
Hub's sync logic is unchanged.

## Hub-side code referenced

| File | Role in the bug |
|---|---|
| `apps/meteor/app/api/server/v1/fileManagement.ts` (delete endpoint) | Hard-deletes both MinIO object and `privos_files` row. No tombstone. |
| `apps/meteor/server/lib/fileManagementSync.ts` (`checkAndSyncFile`, `syncFolder`) | Treats MinIO as source of truth. If the object exists, the row is (re-)inserted. This is what made the resurrection visible to the user. |
| `apps/meteor/app/api/server/v1/internal/rooms/files.ts` (sync endpoint) | Trigger the sandbox calls after a push to refresh Hub's view of MinIO. |

None of these required changes for the immediate bug fix. The asymmetry
that caused the resurrection was on the sandbox side; Hub merely reflected
what MinIO showed.

## Activation

The sandbox-side fix is gated behind an env var so it can be rolled out
safely:

```
RECYCLEBIN_ENABLE_PULL_PRUNE_MAIN=1   # set on privos-sandbox processes
```

Without it: recyclebin code paths run (pre-overwrite captures only — never
deletes anything), but the destructive policy change stays off and the
bug persists. With it: pull queues `delete-local` for main-folder files
that have a `done` row in the sandbox's `file_sync_state`, archives them
to the recyclebin, and unlinks. Push then has nothing to re-upload and
the deletion sticks.

The gate is a sandbox env var, not a Hub setting. Hub admins requesting
the fix should ask the sandbox operator to flip it.

## Defense in depth (optional, not yet implemented)

The recyclebin protects against the resurrecting-pusher (the sandbox).
But Hub still has no way to defend against any *other* future re-uploader
— a manual `mc cp`, a backup restore, or an old sandbox build that hasn't
deployed the gate. A defense-in-depth measure on the Hub side would be a
**deletion tombstone**:

- On `file-management.files/:fileId` DELETE, set a `deletedAt` field
  instead of hard-deleting (or write to `privos_archive_files`).
- In `fileManagementSync.ts`, skip insert/upsert when a tombstone matches
  the `file_path`.
- Background job: after N days, hard-delete tombstones AND re-issue a
  MinIO delete (in case a sandbox repushed it).

This is not built. It's listed here so the option is documented if the bug
ever recurs in production despite the recyclebin being deployed.

## Restore

The sandbox's `Recyclebin.restore()` exists in code but has no HTTP
endpoint or UI yet. The intended end-to-end flow:

1. User clicks "Restore" in the Hub file-management UI on a tombstoned
   row.
2. Hub clears its tombstone (if/when tombstones land per the previous
   section).
3. Hub calls a sandbox endpoint that runs `Recyclebin.restore(id, destPath)`.
4. The next push sends the restored bytes to MinIO.
5. Hub's sync re-inserts the row; UI shows the file again.

Until both the tombstone and the restore endpoint are built, the
recyclebin entries are recoverable only by a sandbox operator with shell
access (see the "Operating" section of the sandbox doc).

## Where to find more

- Full design + implementation:
  [`privos-sandbox/docs/features/recyclebin.md`](../../../privos-sandbox/docs/features/recyclebin.md)
- Recyclebin module source: `privos-sandbox/src/lib/recyclebin/`
- Pull-side trigger wiring: `privos-sandbox/src/lib/minio-pull-queue.ts`
- GC scheduling: `privos-sandbox/src/lib/minio-push-queue.ts`
