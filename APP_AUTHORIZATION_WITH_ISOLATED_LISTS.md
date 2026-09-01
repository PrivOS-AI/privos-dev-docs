# Building app-level authorization on isolated-list data

> **For AI coding agents / assistants:** this file is the authoritative spec for adding per-record
> read/write authorization to a PrivOS MCP app. Read it end to end before writing code. Do exactly
> what the **Rules (MUST / MUST NOT)** section says and follow the **Implementation checklist**; do
> not invent your own permission storage, roles table, or ACL engine — the Hub already enforces this
> model. The exact tool argument shapes are in the tables below (they are the contract); the runnable
> reference is the `privos-mcp-app-demo` **Custom Permissions** tab (`src/ui/custom-permissions-panel.tsx`).
> If a rule here conflicts with older docs, this file wins.

A guide for MCP app builders (human or AI): how to give your app a real per-record authorization model —
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

## Rules (MUST / MUST NOT)

These are enforced by the Hub; your implementation must respect them or it will get runtime denials.

- **MUST** store records that need per-record ACL on a list created with `isolatedList: true`. On a
  non-isolated list, `additionalReaders`/`additionalEditors` are **inert** (the item is already
  room-readable) — do not rely on them there.
- **MUST** do all reads/writes on the current user's session via the `mcpapp.*` tools (the mediated
  `execution: 'user'` path). Your app acts AS the user and can never exceed that user's Lists-UI
  powers — this is the security guarantee.
- **MUST** treat setting `additionalReaders` / `additionalEditors` / `ASSIGNEE` on an isolated item
  as **room owner/admin only** (Hub policy `checkItemGrantFieldWrite`, raw role check). Gate the
  "grant access" affordance on the current user being owner/admin, and surface the Hub's denial
  politely when they are not.
- **MUST** use only permission ids that exist in the room's catalog (`...customPermissions.list`);
  an unknown id is rejected.
- **MUST** put a read-only grant in `additionalReaders` and a read+write grant in `additionalEditors`
  (do not also list the same id in `additionalReaders` when it is already an editor).
- **MUST NOT** build your own roles table, permission store, or ACL check — read/enforce through the
  Hub tools; it is the source of truth and applies grants at every chokepoint (incl. revocation).
- **MUST NOT** try to assign a custom permission to a bot or `app` principal — roles are granted to
  **human members only**.
- **MUST NOT** route per-user reads/writes through internal service-key endpoints — those bypass the
  per-user ACL by design.
- **MUST NOT** rely on an AI agent to set grants or edit items *as a user* — grant management is a
  human owner/admin action (see "Limits to design for").

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

## Implementation checklist (agent-actionable)

When adding this to an app, do these in order:

1. **Declare scopes** in `privos-app.json` `permissions[]`: `lists:read` (+ `lists:write` if the app
   creates/edits records), and `custom-permissions:read` / `custom-permissions:write` if the app shows
   or sets grants. Give each a `reason` + `degradedBehavior`. Re-run `npm run build` (regenerates +
   lints the manifest) after any manifest change.
2. **Model records as isolated-list items** (`mcpapp.lists.create { isolatedList: true }`), not a
   custom store — you get the row-level ACL for free.
3. **Read via the user session** with `mcpapp.lists.getItems` / `mcpapp.lists.query*` — the returned
   set is already filtered to what the current user may see; render exactly that (a shorter list is
   correct, not an error).
4. **Show roles + access** (optional) with `mcpapp.rooms.customPermissions.list` (any member) and
   `...members` (owner/admin only → degrade to role names without member identities otherwise).
5. **Grant/revoke** with `mcpapp.rooms.customPermissions.setItemAccess { itemId, additionalReaders?,
   additionalEditors? }`, gated in the UI on the current user being room owner/admin; wrap it so a Hub
   denial (non-owner) is shown as a clear message, not a crash.
6. **Do not** create/assign permissions from the app — those are human owner/admin REST actions
   (`rooms.customPermissions.create` / `.assign`).
7. **Handle the optional-scope-absent case**: if `custom-permissions:*` is not granted, disable the
   grant UI with the declared degraded behavior rather than calling the tool.

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
