# Building app-level authorization on isolated-list data

A guide for MCP app builders: how to give your app a real per-record authorization model —
"who may read this record, who may edit it" — **without** writing your own permission engine, by
reusing two platform primitives:

1. **Isolated lists** — a list whose items are private by default (visible only to their creator,
   their assignee, and the room owner/admin), so your app's records are not room-wide-readable.
2. **Room custom permissions + the item `additionalReaders` / `additionalEditors` fields**
   ("Readable" / "Editable" grants) — named, owner-managed roles you attach to individual records
   to widen read or read+write beyond the defaults.

The Hub enforces all of it at every read/write chokepoint, so a correct app is just declaring the
right scopes and calling the built-in tools **as the user**. Reference for the exact rules:
[`ROOM_CUSTOM_PERMISSIONS.md`](ROOM_CUSTOM_PERMISSIONS.md). Worked, runnable example: the **Custom
Permissions** tab in `privos-mcp-app-demo` (`src/ui/custom-permissions-panel.tsx`).

---

## The mental model

Think of an isolated list as a table whose row-level ACL the Hub computes for you:

| Capability | Who gets it |
|---|---|
| **READ** a row | creator ∪ assignee (`ASSIGNEE` field) ∪ room owner/admin ∪ holder of any permission id in the row's `additionalReaders` **or** `additionalEditors` |
| **WRITE** a row (edit / move / reorder / delete) | creator ∪ assignee ∪ room owner/admin ∪ holder of any permission id in `additionalEditors` |

- `additionalReaders` = **Readable** grant (read only). `additionalEditors` = **Editable** grant
  (read + write). Editable implies readable; do not also list an id in `additionalReaders`.
- **Read cascades to descendants** through the parent chain (a grant on an epic reveals its
  sub-tree). **Write does not cascade** — each sub-item needs its own `additionalEditors` grant.
- A read grant **never** confers write, delete, or move. Deletes authorize each target by write
  authority.
- On a **non-isolated** list these fields are **inert** (the item is already room-readable), so this
  model only means anything on `isolatedList: true`.

A "role" is a **room custom permission**: a named label (e.g. `Legal`, `Reviewer`) the room
owner/admin defines and assigns to **human members**. You grant a *record* to a *role* by putting
the permission's id into the record's `additionalReaders` / `additionalEditors`. Anyone holding
that permission then reads/edits the record — membership changes take effect immediately at every
chokepoint (revoking the permission drops the access).

---

## Who is allowed to do what (the invariants you must design around)

- **Only a room owner/admin may set `additionalReaders` / `additionalEditors` or the `ASSIGNEE`
  field on an isolated-list item.** Enforced by one Hub policy (`checkItemGrantFieldWrite`) on every
  write path, via a raw owner/admin role check — a plain member (or a DM participant) is refused.
  So the "grant access" actions in your UI only work when the current user is an owner/admin; design
  the affordance to appear/enable accordingly and surface the Hub's denial politely otherwise.
- **Custom permissions are assigned to humans only.** Bot and `app` principals cannot be *granted*
  a permission (be a holder). This model is for human role-based access, not for widening a bot's reach.
- **Every permission id must exist in the room's catalog.** Setting a grant to an unknown id is
  rejected.
- Your app acts **as the current user** (see next section), so it can never exceed what that user
  could do in the Lists UI — an app installed by a member simply gets "permission denied" on the
  owner-only steps. That is the security guarantee, not a limitation to work around.

---

## How your app calls it — always as the user

Do the reads and writes on the **user's own session** through the mediated tool-call path, so the
Hub runs them as that user (`execution: 'user'`) with the user's exact ACL. In the app UI:

```ts
// via @privos_ai/app-react — usePrivosApp().callTool(name, args) posts to
// /api/v1/mcp-apps.tool-call on the user's session; the Hub authorizes + attributes as the user.
const catalog = await callTool('mcpapp.rooms.customPermissions.list', { roomId });
```

Scopes to declare in `privos-app.json` (`permissions[]`):

| Scope | Lets the app | Tools |
|---|---|---|
| `lists:read` | read isolated items the user may see | `mcpapp.lists.getItems`, `mcpapp.lists.get*` |
| `lists:write` | create lists/fields/items, edit items | `mcpapp.lists.create` / `addField` / `createItem` / `updateCustomField` |
| `lists:query` | page/search without reading the whole list | `mcpapp.lists.query*` |
| `custom-permissions:read` | read the room catalog + a permission's holders | `mcpapp.rooms.customPermissions.list` `{roomId?}`, `mcpapp.rooms.customPermissions.members` `{roomId?, permissionId}` |
| `custom-permissions:write` | set a record's Readable/Editable grants | `mcpapp.rooms.customPermissions.setItemAccess` `{itemId, additionalReaders?, additionalEditors?}` |

Note: **minting** custom permissions and **assigning** them to members are *human owner/admin*
actions via the current-user REST endpoints (`rooms.customPermissions.create` / `.assign`) — they
are deliberately **not** exposed as MCP tools. An app reads the catalog and sets item grants; it
never creates roles or puts people into them. `setItemAccess` also re-checks owner/admin on the Hub
regardless of the granted scope.

---

## End-to-end recipe (matches the demo tab step for step)

1. **Define a role** (owner/admin, REST): `POST rooms.customPermissions.create { roomId, name: "Legal" }`.
2. **Put people in the role** (owner/admin, REST): `POST rooms.customPermissions.assign { roomId, permissionId, userId }`.
3. **Store your data on an isolated list** (owner/admin for the create): `mcpapp.lists.create
   { roomId, name, isolatedList: true, ... }` → `mcpapp.lists.createItem { listId, title, ... }`.
4. **Grant a record to the role** (owner/admin): `mcpapp.rooms.customPermissions.setItemAccess
   { itemId, additionalReaders: [permissionId] }` (or `additionalEditors` for read+write).
5. **Everyone reads through their own ACL**: any holder of `Legal` now sees the item via
   `mcpapp.lists.getItems`; a non-holder, non-creator, non-assignee member does not. Revoke the
   permission from a user and their access disappears on the next read.

To *inspect* access ("who can read/edit this record, what roles exist here?"), read the catalog
(`.list`, any member) and the holders (`.members`, owner/admin only). Design the "who has access"
view to degrade to role names without member identities when the viewer is not owner/admin.

---

## Verifying it works (two/three accounts, real Hub)

1. Account **A** (owner/admin): run steps 1–4, granting `Legal` to account **B** and setting
   `additionalReaders: [Legal]` on the record.
2. Account **B** (holds `Legal`): opens the list → sees the record (read grant).
3. Account **C** (member, not creator/assignee/owner-admin, no `Legal`): does **not** see it.
4. Owner revokes `Legal` from B → B loses the record on the next read.
5. Swap `additionalReaders` for `additionalEditors` → B can now edit; C still cannot.

---

## Limits to design for

- **No agent write-on-behalf.** An AI agent cannot set these grants (or edit items) *as a user*;
  grant management is a human owner/admin action in a UI session. See
  [`ROOM_CUSTOM_PERMISSIONS.md`](ROOM_CUSTOM_PERMISSIONS.md) → "Limitation — no agent write-on-behalf".
  (An agent bot deliberately given the room owner/admin role can write **as itself** — a separate,
  config-gated choice.)
- **The internal service-key routes bypass per-user ACL by design** — never route your app's
  per-user reads/writes through them; use the mediated `mcpapp.*` tools shown above.
- **Grant ids are metadata visible to any reader of the item** (permission names are member-readable
  via the catalog anyway); do not treat the id list itself as secret.
