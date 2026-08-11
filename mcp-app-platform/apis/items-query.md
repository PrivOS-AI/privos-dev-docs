# `items.query` — filtered, paginated item retrieval

`POST /api/v1/items.query`

Returns a window of a list's items, chosen by a structured filter and paged by an
opaque cursor. Use it instead of `items.listByListId`, which answers with at most 500
items and cannot filter.

**Scope:** `lists:query`. Declare it in `privos-app.json`; it is separate from
`lists:read` because it is a POST, and the platform keeps every `*:read` scope GET-only.

---

## Request

```jsonc
{
  "listId": "L1",
  "filter": {
    "stageId": "S1",
    "parentId": null,              // null = top-level only; "<id>" = children of it; absent = any depth
    "archived": false,
    "text": "invoice",
    "createdAt": { "gte": "2026-08-01T00:00:00Z", "lte": "2026-08-06T00:00:00Z" },
    "updatedAt": { "gte": "2026-08-05T12:00:00Z" },
    "customFields": [
      { "fieldId": "f1", "op": "contains", "value": "acme" },
      { "fieldId": "f2", "op": "greater_than_or_equal", "value": 5 }
    ]
  },
  "sort":   { "field": "order", "direction": 1 },
  "cursor": "<opaque>",
  "count":  20,
  "fields": ["name", "key", "stageId", "order", "customFields"]
}
```

| Field | Notes |
|---|---|
| `listId` | Required. The caller must be a member of the list's room. |
| `filter` | Optional. Unknown keys are rejected. |
| `sort` | `field` is one of `order`, `createdAt`, `_updatedAt`, `name`; `direction` is `1` or `-1`. Defaults to `order` ascending. |
| `cursor` | A previous response's `nextCursor`, passed back **verbatim**. |
| `count` | Default 20, maximum 200. |
| `fields` | Trims the response only. The server always reads what it needs to apply visibility rules. |

Request bodies are capped at **16 KB** and the endpoint is rate limited to **120
requests per minute per user**.

## Response

```jsonc
{
  "items": [ /* ... */ ],
  "count": 20,
  "nextCursor": "<opaque>",   // null when the list is exhausted
  "success": true
}
```

**There is no total.** Counting a list is proportional to its size, and on an isolated
list a total would describe rows the caller is not permitted to see. Page until
`nextCursor` is `null`.

## Field conditions

`customFields` accepts custom field ids and these system field ids: `name`,
`description`, `key`, `createdBy`, `createdAt`, `_updatedAt`. Conditions are ANDed.

| Operator | Applies to |
|---|---|
| `contains`, `does_not_contain`, `starts_with`, `ends_with` | text-like fields, file names |
| `is`, `is_not` | every type (equality; `equals`/`does_not_equal` are accepted aliases) |
| `greater_than`, `less_than`, `greater_than_or_equal`, `less_than_or_equal` | numbers |
| `is_before`, `is_after`, `is_on_or_before`, `is_on_or_after` | dates, date-times, deadlines |
| `is_empty`, `is_not_empty` | every type; no `value` |

Values are scalars, or arrays of scalars for multi-valued fields. **Objects are
rejected** — pass a user's id, not a user object. A condition whose `value` is empty is
dropped rather than matching nothing, which mirrors the board's own filters.

Two inherited behaviours are worth knowing:

- an unset number reads as `0`, so `less_than 4` matches items that never had the field set;
- `is_empty` and `is_not_empty` are not opposites: `0` and `false` satisfy both.

## Cursors

A cursor encodes the sort it was issued under. Change `sort` and the cursor is
rejected with `error-invalid-cursor` — start the walk again. Cursors are short-lived
tokens, not bookmarks; do not store one.

After a create, delete, reorder or stage move, discard the cursor and re-page. Ranks are
renumbered by those operations, so a cursor issued before one can skip or repeat rows.
**Always merge pages by `_id`.**

## Free-text search

`filter.text` matches an item's name, key, description, its creator's username, its
stage's name, and the rendered value of its custom fields — option labels for choice
fields, usernames for people fields, file names for attachments, and the name or key of
a referenced item for dependency fields.

Not searched: date fields (their rendered form depends on the reader's locale and
timezone) and recurrence rules.

Results are ordered by the requested sort, never by relevance. For ranked search use
`items.search`.

---

## Examples

### One window of a stage

```bash
curl -X POST "$HUB/api/v1/items.query" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $USER_ID" \
  -H 'Content-Type: application/json' \
  -d '{"listId":"L1","filter":{"stageId":"S1","archived":false},"count":20}'
```

### Filter on a custom field, ask for three fields

```tsx
const page = await app.rest({
  method: 'POST',
  path: 'items.query',
  body: {
    listId,
    filter: { customFields: [{ fieldId: 'amount', op: 'greater_than', value: 1000 }] },
    sort: { field: 'createdAt', direction: -1 },
    fields: ['name', 'key', 'customFields'],
  },
});
```

### Walk every page

```tsx
const all = new Map<string, Item>();
let cursor: string | undefined;

do {
  const res = await app.rest({ method: 'POST', path: 'items.query', body: { listId, count: 100, cursor } });
  res.body.items.forEach((item) => all.set(item._id, item));   // merge by _id
  cursor = res.body.nextCursor ?? undefined;
} while (cursor);
```

### Incremental sync

Record when a sync finished, then ask only for what changed since:

```tsx
const since = localStorage.getItem('lastSync') ?? new Date(0).toISOString();

const res = await app.rest({
  method: 'POST',
  path: 'items.query',
  body: { listId, filter: { updatedAt: { gte: since } }, sort: { field: '_updatedAt', direction: -1 }, count: 100 },
});

res.body.items.forEach((item) => cache.set(item._id, item));
localStorage.setItem('lastSync', new Date().toISOString());
```

Deletions do not appear in this stream. Re-page in full occasionally if your copy must
converge exactly.

## Errors

| Error | Cause |
|---|---|
| `error-invalid-params` | `listId` missing or not a string |
| `error-list-not-found` | No such list |
| `error-invalid-filter` | Unknown filter key, unknown `fieldId`, or unrecognised operator |
| `error-invalid-filter-value` | Value is an object, an over-long string, an over-long array, or begins with `$` |
| `error-invalid-cursor` | Cursor is malformed, tampered with, or was issued for a different sort |
| `error-invalid-sort` | Sort field outside the allowed set |
| `error-invalid-count` | `count` is not a positive integer |
| `error-invalid-fields` | A requested field is not selectable |
| `error-request-too-large` | Body over 16 KB |

## Isolated lists

On a list marked isolated, a member sees only items they created or are assigned to,
plus the sub-items of those. Windows stay full size — the server keeps reading until it
has `count` permitted items — and no total is exposed.

## MCP tool equivalent

`mcpapp.lists.queryItems` takes the same arguments and requires the same scope.
`mcpapp.lists.getItems` remains for compatibility, but pages by offset, so rows shift
across page boundaries whenever the list changes between calls.
