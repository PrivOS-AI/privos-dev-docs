# Room Custom Permissions & Item Access Grants

Per-room **custom permissions** and two additive item fields — **`additionalReaders`** and
**`additionalEditors`** — that extend read/edit on **isolated-list** items to permission holders
beyond the item's creator, assignee, and the room owner/admin.

## Concept

- A **room custom permission** is a named label a room owner/admin defines per room and assigns
  to **human members** (bot / `app` principals are refused). Assignment is stored per member.
- An **isolated-list item** carries two optional fields:
  - `additionalReaders: string[]` — custom-permission ids whose holders may **READ** the item.
  - `additionalEditors: string[]` — custom-permission ids whose holders may **READ and EDIT** it.
- Both are **additive grants**. On a **non-isolated** list they are inert (the item is already
  room-readable); they never restrict.

### Resolution

| Capability | Who |
|---|---|
| READ | creator ∪ assignee ∪ owner/admin ∪ holder of any id in `additionalReaders` **or** `additionalEditors` |
| WRITE (edit / move / reorder / delete) | creator ∪ assignee ∪ owner/admin ∪ holder of any id in `additionalEditors` |

- Read grants **cascade to descendants** through the parent chain, like assignee visibility.
  Write does **not** cascade — each sub-item needs direct write authority.
- A read grant never confers write, delete, move, or file-write. Batch deletes authorize each
  target by write authority.
- Item comment rooms and item files follow READ.

## REST endpoints (current user)

Base: `/api/v1/rooms.customPermissions.*`, `authRequired: true`. Managing is **room owner/admin
only** (raw role check — a DM participant is not owner/admin); listing is any room member;
assignment is **human members only**.

| Method | Endpoint | Body / query | Auth |
|---|---|---|---|
| GET | `rooms.customPermissions.list` | `{ roomId }` | any member |
| POST | `rooms.customPermissions.create` | `{ roomId, name, description? }` | owner/admin |
| POST | `rooms.customPermissions.update` | `{ roomId, permissionId, name?, description? }` | owner/admin |
| POST | `rooms.customPermissions.delete` | `{ roomId, permissionId }` | owner/admin |
| POST | `rooms.customPermissions.assign` | `{ roomId, userId, permissionId }` | owner/admin, target human |
| POST | `rooms.customPermissions.unassign` | `{ roomId, userId, permissionId }` | owner/admin |
| GET | `rooms.customPermissions.assignments` | `{ roomId, userId? }` | owner/admin (or self) |
| GET | `rooms.customPermissions.members` | `{ roomId, permissionId }` | owner/admin |

`create`/`update` enforce a unique `(roomId, name)`. `delete` unassigns everyone and reclaims
comment-room access (see Revocation). Errors: `error-not-allowed`, `error-duplicate-name`,
`error-invalid-principal` (non-human target), `error-not-found`, `error-invalid-params`.

### Setting item grants

The two item fields are written through the normal item update — **owner/admin only** on an
isolated list, and each id must exist in the item's room catalog:

```http
POST /api/v1/items.update
{ "itemId": "…", "additionalReaders": ["<permId>"], "additionalEditors": ["<permId>"] }
```

`null` clears a field (stored as `[]`). The same owner/admin gate protects writing the ASSIGNEE
field on an isolated list (closing self-assign-to-gain-read).

## MCP scopes & tools

Catalog version `2026-08-31`. Two scopes let a marketplace app work with grants in an approved
room (declare them in `privos-app.json` `permissions`):

| Scope | Risk | Tools |
|---|---|---|
| `custom-permissions:read` | medium (personal) | `mcpapp.rooms.customPermissions.list` `{ roomId? }` → `[{_id,name,description?}]`; `mcpapp.rooms.customPermissions.members` `{ roomId?, permissionId }` → `[{_id,username,name}]` |
| `custom-permissions:write` | high (personal) | `mcpapp.rooms.customPermissions.setItemAccess` `{ itemId, additionalReaders?, additionalEditors? }` → `{ updated, itemId }` |

- `setItemAccess` still requires the acting principal to be a **room owner/admin** and every id to
  exist in the item's room catalog; the item's list must live in the approved room.
- Catalog CRUD and member assignment are **not** exposed as MCP tools — they are human owner/admin
  actions via the REST endpoints above. An app reads the catalog and sets item access; it never
  mints permissions or assigns them to people.

## Rollout — `Isolated_Item_Write_ACL_Enforce`

Per-item WRITE enforcement is a behavior change (REST/internal routes had no per-item check;
MCP allowed any read-visible caller). It ships **log-only by default**: would-be denials log
`[isolated-item-write-acl] would-deny path=… item=… user=…` and the write proceeds. Flip the
setting on (admin settings) after the logs show no legitimate violators. Confidentiality gates
(reads, comment-room open, grant-field / ASSIGNEE writes, human-only assignment) are always
enforced regardless of the flag.

## Revocation

Grants are dynamic: removing a permission from a user (or deleting it) drops their read/edit at
every chokepoint immediately, and an async sweep removes their subscriptions to item comment rooms
they can no longer read; deleting a permission also pulls the dangling id from item grant arrays.
Residual: already-issued presigned file URLs live until their TTL.

## Limitation — no agent write-on-behalf

Distinguish two cases:

- **As a user (impersonation) — reads yes, writes no.** An agent can **read** items honoring these
  grants under a user's ACL in three places: the user's own app-UI session, the private "on my behalf"
  assistant session, and — behind `Agent_Delegated_Read_Enabled` — the room bot in the AI Chat window
  of a dedicated-mode room ([`DELEGATED_READ_AS_ASKER.md`](DELEGATED_READ_AS_ASKER.md)). The
  per-attempt read grant is GET-only by construction: any write carrying it is refused with
  `delegated-read-write-denied`. An **autonomous agent cannot write on a user's behalf** — it cannot
  set `additionalReaders`/`additionalEditors`, reassign, edit, or delete *as the user*; writes stay a
  human action.
- **As itself (its own bot identity) — supported, config-gated.** `setItemAccess` and the item write
  gates authorize against the **acting principal** via a room-role check. So if a room owner
  deliberately gives the agent bot the room `owner`/`admin` role, the bot **can** set grants, reassign,
  and edit/delete isolated items — attributed to the bot, scoped to the authority granted. Bound the
  trigger surface by controlling room membership. (A bot can never be *assigned* a custom permission as
  a grantee — that is human-only; this is about the bot holding a room role.)

## See also

- [`APP_AUTHORIZATION_WITH_ISOLATED_LISTS.md`](APP_AUTHORIZATION_WITH_ISOLATED_LISTS.md) — builder
  guide: how an MCP app uses isolated lists + these Readable/Editable grants as its own per-record
  authorization model (with the `privos-mcp-app-demo` Custom Permissions tab as the worked example).
- [`PRIVATE_AI_ON_BEHALF.md`](PRIVATE_AI_ON_BEHALF.md) — the private "on my behalf" AI-chat session
  reads a shared room's items under the member's own ACL (honoring these grants); reads only.
- [`DELEGATED_READ_AS_ASKER.md`](DELEGATED_READ_AS_ASKER.md) — the sandbox room bot reading isolated
  lists as the asker in the AI Chat window (dedicated rooms, flag-gated, reads only).
- [`room-scoped-apis/items.md`](room-scoped-apis/items.md) — the internal item routes now filter
  isolated items by visibility and gate writes.
- [`MCP_APP_PLATFORM.md`](MCP_APP_PLATFORM.md) — scope declaration + mediated tool calls.
