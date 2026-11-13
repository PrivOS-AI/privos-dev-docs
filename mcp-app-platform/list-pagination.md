# Paging MCP list tools (`paginate: true`)

Every built-in MCP tool that returns a list can answer with one common envelope: the items plus the information needed to
read the next page. You ask for it with `paginate: true`. Without that flag a tool keeps the response shape it has always
had, so existing apps keep working unchanged.

**Availability:** the **Production version** and the **Develop version**.

**The envelope is opt-in for now, and becomes the default on 2026-11-20.** Migrate your list calls before then: after
the switch the old shapes are no longer returned, so an unmigrated call either throws (`.map` / `.find` on an object)
or silently reads nothing (`Array.isArray(result) ? result : []`).

## The envelope

```json
{
  "items": [ ... ],
  "pagination": {
    "limit": 50,
    "offset": 100,
    "total": 1234,
    "hasMore": true,
    "nextOffset": 150,
    "nextCursor": null
  }
}
```

| Field | Meaning |
|---|---|
| `items` | The rows of this page, in the tool's usual row shape. |
| `limit` | The page size that was applied. |
| `offset` | How many rows were skipped. `null` on a cursor page of `lists.queryItems`. |
| `total` | How many rows match in all. `null` when it cannot be given (see below). |
| `hasMore` | `true` when at least one more row exists. Exact on every tool (each page reads one row past `limit`), except `mcpapp.db.query` with `totalCapped: true`, where it means "maybe more" — see below. |
| `nextOffset` | The `offset` of the next page, or `null` on the last page. |
| `nextCursor` | `lists.queryItems` only: the cursor of the next page, or `null`. |
| `totalCapped` | Present and `true` only on `mcpapp.db.query` when more than 10,000 records match. |

## Arguments

| Argument | Default | Notes |
|---|---|---|
| `paginate` | — | Must be `true` to get the envelope. |
| `limit` | 50 | Maximum 200. A larger value is lowered to 200. Must be a positive integer. |
| `offset` | 0 | Must be a non-negative integer. |

Older argument names still work: `count` for `lists.getItems` and `lists.queryItems`, `skip` for `files.getByChannel` and
`folders.getByChannel`. A non-integer or negative `limit` / `offset` is rejected.

## When `total` is `null`

`hasMore` and `nextOffset` are always correct. `total` is `null` in two cases:

- **Isolated lists.** A member sees only some items of an isolated list, and a count would reveal how many items they
  cannot see. Room owners and admins see every item and get the real total. An offset page that reaches the end of
  the list knows the total and returns it, even for a member.
- **Slow counts.** A count that runs longer than 10 seconds is stopped and `total` is `null`.

`mcpapp.db.query` counts at most 10,000 records. Above that, `total` is 10,000 and `totalCapped` is `true`, because a
filter on an app-defined field may have no index.

`mcpapp.db.query` derives `hasMore` from `total`, so with `totalCapped: true` it can no longer tell whether rows remain
and errs on the safe side: a full page is reported as `hasMore: true`. A walk then spends one extra call that returns
`items: []` and ends. It never errs the other way, so no rows are lost — a page shorter than `limit` always means the
end. Every other tool reads one row past `limit`, so their `hasMore` is exact.

## Tools that accept `paginate`

| Tool | Notes |
|---|---|
| `mcpapp.lists.getAll` | |
| `mcpapp.lists.getItems` | Same filters as before (`stageId`, `customFieldFilters`, `sortBy`, `sortOrder`). |
| `mcpapp.lists.queryItems` | By offset, or by `cursor` when you pass one (then `offset` is `null`; follow `nextCursor`). |
| `mcpapp.lists.getItemsByStage` | |
| `mcpapp.lists.getItemsByStages` | Answers `{ [stageId]: { items, pagination } }`; `limit` and `offset` apply per stage. |
| `mcpapp.lists.searchItems` | |
| `mcpapp.lists.getSubItems` | |
| `mcpapp.stages.getByList` | |
| `mcpapp.files.getByChannel` | |
| `mcpapp.files.search` | Matches from incoming shared folders come after the room's own files. |
| `mcpapp.folders.getByChannel` | At the root, the synthetic `Shared-Folders` entry comes after every real folder. |
| `mcpapp.folders.getRootContent` | Answers `{ childFolders, childFiles }`, each `{ items, pagination }`; root files are included. |
| `mcpapp.folders.getContent` | Answers `{ folder, childFolders, childFiles }`, the two lists paged separately. |
| `mcpapp.folders.search` | |
| `mcpapp.messages.getRecent` | Newest first; `offset` counts back from the latest message. |
| `mcpapp.rooms.getMembers` | |
| `mcpapp.rooms.customPermissions.list` | |
| `mcpapp.rooms.customPermissions.members` | |
| `mcpapp.db.query` | Rows are sorted by `orderBy`, then `_id`, so pages never repeat or skip a record. |
| `mcpapp.db.listCollections` | |

## Reading every page

```ts
async function readAll(app, name, args) {
  const rows = [];
  let offset = 0;
  for (;;) {
    const page = await app.callServerTool({ name, arguments: { ...args, paginate: true, limit: 200, offset } });
    const body = JSON.parse(page.content[0].text);
    rows.push(...body.items);
    if (!body.pagination.hasMore) return rows;
    offset = body.pagination.nextOffset;
  }
}

const files = await readAll(app, 'mcpapp.files.getByChannel', { channelId: roomId });
```

For a long walk over list items, prefer the cursor of `lists.queryItems`: an offset counts rows from the start on every
call, so a deep offset is both slow and eventually refused.

- `lists.getItems` and `lists.queryItems` (in offset mode) raise *"Offset is too deep for this list; page with a cursor
  or narrow the filter"*.
- On an **isolated** list, `lists.getItemsByStage`, `lists.getItemsByStages`, `lists.searchItems` and
  `lists.getSubItems` raise *"Offset is too deep for this isolated list; narrow the query"* once the visibility walk has
  scanned its bound. On a normal list these four page in the database and have no such limit.

The cursor resumes where the last page ended and stays stable while the list changes.

```ts
let cursor;
do {
  const page = await app.callServerTool({
    name: 'mcpapp.lists.queryItems',
    arguments: { listId, paginate: true, limit: 200, ...(cursor ? { cursor } : {}) },
  });
  const body = JSON.parse(page.content[0].text);
  handle(body.items);
  cursor = body.pagination.nextCursor;
} while (cursor);
```

## Moving an existing call to the envelope

The response shape changes only when you add `paginate: true`, so move one call at a time and change its reader in the
same commit:

| Tool | Legacy response | With `paginate: true` |
|---|---|---|
| Most list tools | a bare array | `body.items` |
| `mcpapp.db.query` | `{ records, total, totalCapped }` | `body.items`, `body.pagination.total` |
| `mcpapp.lists.getItems` | `{ items, count, offset }` | `body.items`, `body.pagination` |
| `mcpapp.lists.queryItems` | `{ items, count, nextCursor }` | `body.items`, `body.pagination.nextCursor` |

Default page sizes also change. Without `paginate`, `lists.getAll`, `stages.getByList` and `lists.getSubItems` return
everything, and the rest cap silently:

| Tool (legacy shape) | Default page | Hard maximum |
|---|---|---|
| `mcpapp.db.query` | 1000 | 1000 — a larger `limit` is lowered, with no error |
| `mcpapp.rooms.getMembers` | 50 | 200 — a larger `limit` is lowered, with no error |
| `mcpapp.lists.getItems` | 50 | 100 |
| `mcpapp.lists.getItemsByStage` | 100 | no maximum |
| `mcpapp.lists.searchItems` | 50 | no maximum |
| `mcpapp.lists.queryItems` | 20 | 200 |
| `mcpapp.messages.getRecent` | 50 | 100 |
| `mcpapp.files.search`, `mcpapp.folders.search` | 20 | no maximum |
| `mcpapp.files.getByChannel`, `mcpapp.folders.getByChannel` | 50 | no maximum |

`lists.queryItems` is the only one of those that does not truncate silently: its legacy shape returns `nextCursor`, so a
caller who follows the cursor reads everything.

With `paginate: true` every tool returns 50 unless you pass `limit` (maximum 200). A caller that expects "all rows in
one call" must loop on `hasMore`.

## Visibility rules on file and folder tools

All file and folder tools, with or without `paginate`, follow the room's Files tab rules:

- Names starting with `.` are hidden unless the user holds `view-room-hidden-files` (default admin, owner, leader). A
  hidden file or folder answers "not found".
- The folder of an item on an isolated List is visible only to users who can see that item. Changing anything inside
  it (rename, move, delete, create a subfolder) needs write access to that item.

Listings apply both rules inside the query, so `total` and `hasMore` count only what the user may see.

## REST

The REST routes have their own paging (`offset` / `count`, and `items.query` with `cursor`). See
[Item Query](./apis/items-query.md) and [Auth & REST Integration](./auth-and-rest-integration.md).
