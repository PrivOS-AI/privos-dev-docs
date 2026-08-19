# PrivOS MCP Tools — Database

## Overview

The `mcpapp.db.*` tools give apps a relational database with schema registration, Firestore-style query builder, typed references, and migration support. `mcpapp.objects.*` (see [App-Owned Objects](#app-owned-objects-content-addressed-storage) below) gives apps a companion content-addressed blob store under the same `db:read`/`db:write` scopes, for content that doesn't fit a typed record.

**Data scope:** Apps choose `scope: 'global' | 'room'` per collection.
- **Global:** `app_{appId}_{collection}` — shared across all rooms
- **Room-scoped:** `app_{appId}_{roomId}_{collection}` — isolated per room

**Isolation:** Collections are namespaced per app. App A cannot access App B's data.

## OAuth Scopes

| Scope | Description |
|-------|-------------|
| `db:read` | Read records from app collections |
| `db:write` | Create/update/delete records |
| `db:schema:read` | Read collection schemas |
| `db:schema:write` | Register/update/drop schemas |

---

## Data Scope: Global vs Room

Every collection is registered with a scope, which decides the physical
collection name the hub stores it under
(`apps/meteor/server/services/mcp-app-db-schema-registry.ts:41-43`). The scope is
fixed at registration time; `scope` accepts `global` or `room` and defaults to
`global` (`mcp-tool-handlers-db-schema.ts:18,57`).

| Scope | Collection name | Use case |
|-------|-----------------|----------|
| `global` (default) | `app_{appId}_{collection}` | Shared data across all rooms (user profiles, settings, catalogs) |
| `room` | `app_{appId}_{roomId}_{collection}` | Per-room isolation (project tasks, team notes, channel polls) |

```jsonc
// Global — same data in every room
{ "name": "products", "fields": [ /* … */ ] }

// Room-scoped — each room gets its own collection
{ "name": "poll_votes", "fields": [ /* … */ ], "scope": "room" }
```

---

## Schema Tools

### `mcpapp.db.registerCollection`

Register a new collection with schema definition.

| | |
|---|---|
| **Scope** | `db:schema:write` |

#### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `collection` | string | Yes | Collection name (alphanumeric + underscores, 1-64 chars) |
| `scope` | enum | No | `'global'` (default) or `'room'` |
| `fields` | array | Yes | Field definitions (see below) |
| `indexes` | array | No | Index definitions |

**Field definition:**

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | Field name |
| `type` | enum | `string`, `number`, `boolean`, `date`, `array`, `reference` |
| `required` | boolean | Require field on insert |
| `maxLength` | number | Max string length |
| `enum` | string[] | Allowed values (string fields) |
| `min` / `max` | number | Number range |
| `refCollection` | string | Target collection for references |
| `onDelete` | enum | `cascade`, `set-null`, `restrict`, `no-action` |

#### Example

```json
{
  "collection": "contacts",
  "scope": "global",
  "fields": [
    { "name": "name", "type": "string", "required": true, "maxLength": 100 },
    { "name": "email", "type": "string" },
    { "name": "age", "type": "number", "min": 0, "max": 150 },
    { "name": "company", "type": "reference", "refCollection": "companies", "onDelete": "set-null" }
  ],
  "indexes": [{ "fields": { "email": 1 }, "unique": true }]
}
```

### `mcpapp.db.updateSchema`

Update field definitions (bumps version, uses `validationLevel: 'moderate'`).

| | |
|---|---|
| **Scope** | `db:schema:write` |

| Arg | Type | Required |
|-----|------|----------|
| `collection` | string | Yes |
| `fields` | array | Yes |

### `mcpapp.db.getSchema` / `mcpapp.db.listCollections`

| | |
|---|---|
| **Scope** | `db:schema:read` |

### `mcpapp.db.dropCollection`

Drops the MongoDB collection and removes the schema document.

| | |
|---|---|
| **Scope** | `db:schema:write` |

---

## CRUD Tools

### `mcpapp.db.create`

| | |
|---|---|
| **Scope** | `db:write` |

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `collection` | string | Yes | Collection name |
| `data` | object | Yes | Record data matching schema |

Records automatically get `_id`, `_createdAt`, `_updatedAt`.

```json
// Request
{ "collection": "contacts", "data": { "name": "Alice", "email": "alice@example.com" } }
// Response
{ "_id": "664f...", "name": "Alice", "email": "alice@example.com", "_createdAt": "...", "_updatedAt": "..." }
```

### `mcpapp.db.createMany`

Batch create (max 100 records).

| Arg | Type | Required |
|-----|------|----------|
| `collection` | string | Yes |
| `records` | array | Yes |

### `mcpapp.db.get`

| Arg | Type | Required |
|-----|------|----------|
| `collection` | string | Yes |
| `id` | string | Yes |

### `mcpapp.db.update`

| Arg | Type | Required |
|-----|------|----------|
| `collection` | string | Yes |
| `id` | string | Yes |
| `data` | object | Yes |

### `mcpapp.db.updateMany`

| Arg | Type | Required |
|-----|------|----------|
| `collection` | string | Yes |
| `where` | array | Yes |
| `data` | object | Yes |

### `mcpapp.db.delete` / `mcpapp.db.deleteMany`

Soft-delete: sets `_deletedAt` timestamp. Cascade rules are enforced.

---

## Query Tools

### `mcpapp.db.query`

| | |
|---|---|
| **Scope** | `db:read` |

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `collection` | string | Yes | Collection name |
| `where` | array | No | Filter clauses |
| `orderBy` | array | No | Sort `[{ field, direction }]` |
| `limit` | number | No | Max 1000 (default) |
| `offset` | number | No | Skip for pagination |
| `populate` | string[] | No | Reference fields to resolve |

**Where clause operators:** `==`, `!=`, `>`, `<`, `>=`, `<=`, `in`, `not-in`, `array-contains`, `array-contains-any`

```json
{
  "collection": "contacts",
  "where": [{ "field": "age", "op": ">=", "value": 18 }],
  "orderBy": [{ "field": "name", "direction": "asc" }],
  "limit": 20
}
```

Response: `{ "records": [...], "total": 42 }`

### `mcpapp.db.count`

Returns `{ "count": number }`.

### `mcpapp.db.aggregate`

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `collection` | string | Yes | |
| `op` | enum | Yes | `count`, `sum`, `avg`, `min`, `max` |
| `field` | string | For sum/avg/min/max | Field to aggregate |
| `where` | array | No | Filter |
| `groupBy` | string | No | Group field |

---

## Reference Tools

### `mcpapp.db.populate`

Resolve reference fields to full documents (1-level deep, max 5 fields, max 1000 records).

| Arg | Type | Required |
|-----|------|----------|
| `collection` | string | Yes |
| `ids` | string[] | Yes |
| `fields` | string[] | Yes |

---

## App-Owned Objects (Content-Addressed Storage)

`mcpapp.objects.*` gives an app a private, room-bound blob store for content the typed
`mcpapp.db.*` records don't fit well (attachments, generated artifacts, evidence files).
Objects are immutable and content-addressed: the key is the object's own SHA-256 digest,
so the same content written twice is a no-op ("exact-adopt"), not a duplicate.

**Isolation:** every object is private to the exact `(app, room)` pair that wrote it —
there is no cross-room or cross-app read path, and no `scope: 'global'` option (unlike
`mcpapp.db.*` collections).

### `mcpapp.objects.put`

| | |
|---|---|
| **Scope** | `db:write` |

| Arg | Type | Required | Description |
|-----|------|----------|--------------|
| `digest` | string | Yes | `sha256:<64 lowercase hex>` — must match the actual SHA-256 of `dataBase64` |
| `dataBase64` | string | Yes | Canonical base64 content, max 32 MiB decoded |
| `mediaType` | string | Yes | MIME type, e.g. `application/pdf` |

Returns `{ digest, size, mediaType, createdAt, adopted }`. `adopted: true` means an object
with this exact digest already existed in this room for this app and was reused as-is (no
new write); `adopted: false` means this call created it. Writing the same digest again
with **different** `mediaType` or size is rejected as a content-addressed metadata
conflict — the digest is a hard identity, not a mutable key.

### `mcpapp.objects.head` / `mcpapp.objects.get`

| | |
|---|---|
| **Scope** | `db:read` |

| Arg | Type | Required |
|-----|------|----------|
| `digest` | string | Yes |

`head` returns metadata only (`{ digest, size, mediaType, createdAt, adopted: true }`, no
content). `get` returns the same metadata plus `dataBase64` — the hub re-verifies the
stored bytes against the digest on every read and fails if they don't match. Reading a
digest this app/room never wrote (or wrote under a different app/room) returns "object not
found" — objects are never visible outside the exact pair that created them.

---

## SDK Usage (React)

There is no dedicated `useAppDb()` hook in `@privos_ai/app-react` — call these
tools directly via `usePrivosApp()` / `usePrivosTool()`. See
[React SDK Reference — No dedicated DB hook](../react-sdk-reference.md#no-dedicated-db-hook).

```tsx
import { usePrivosApp, usePrivosTool } from '@privos_ai/app-react';

function MyApp() {
  const app = usePrivosApp();

  // Register schema (call once on first load)
  await app.callServerTool({
    name: 'mcpapp.db.registerCollection',
    arguments: {
      collection: 'contacts',
      fields: [
        { name: 'name', type: 'string', required: true },
        { name: 'email', type: 'string' },
      ],
    },
  });

  // Create / update / delete — all writes go through callServerTool
  const contact = await app.callServerTool({ name: 'mcpapp.db.create', arguments: { collection: 'contacts', data: { name: 'Alice' } } });
  await app.callServerTool({ name: 'mcpapp.db.update', arguments: { collection: 'contacts', id: contact._id, data: { name: 'Bob' } } });
  await app.callServerTool({ name: 'mcpapp.db.delete', arguments: { collection: 'contacts', id: contact._id } });

  // Query — flat where/orderBy args, no chainable builder client-side; usePrivosTool auto-fetches
  const { data, loading, error, refetch } = usePrivosTool('mcpapp.db.query', {
    collection: 'contacts',
    where: [{ field: 'name', op: '!=', value: '' }],
    orderBy: [{ field: 'name', direction: 'asc' }],
    limit: 10,
  });
}
```

---

## Getting Started Tutorial

Complete walkthrough: building a "Feedback Board" app using the DB layer.

> **`db` below is illustrative, not an SDK export.** `@privos_ai/app-react` has no
> `useAppDb()` hook and no chainable query builder — see
> [SDK Usage (React)](#sdk-usage-react) above for the real, non-chainable call
> shape. Every `db.xxx(...)` call in this tutorial (including the chained
> `.where().orderBy().execute()` query form) is shorthand for a single
> `mcpapp.db.*` tool call with flat arguments; write your own thin wrapper
> over `usePrivosApp().callServerTool()` / `usePrivosTool()` if you want this
> ergonomics, or call the tools directly.

### 1. Request Scopes

In your app manifest (`/.well-known/mcp/manifest.json`):

```json
{
  "name": "com.example.feedback",
  "version": "1.0.0",
  "title": "Feedback Board",
  "scopes": ["db:read", "db:write", "db:schema:read", "db:schema:write"]
}
```

### 2. Register Schema on Mount

```tsx
import { useEffect, useRef, useState } from 'react';
// `db` here is a hand-rolled wrapper you write yourself (see the disclaimer
// above) — not an import from '@privos_ai/app-react'.

function FeedbackApp() {
  const db = useDb(); // your own wrapper over usePrivosApp()
  const initialized = useRef(false);

  useEffect(() => {
    if (initialized.current) return;
    initialized.current = true;
    initSchema();
  }, []);

  async function initSchema() {
    try {
      await db.registerCollection('feedback', [
        { name: 'title', type: 'string', required: true, maxLength: 200 },
        { name: 'body', type: 'string', maxLength: 2000 },
        { name: 'category', type: 'string', enum: ['bug', 'feature', 'improvement'] },
        { name: 'votes', type: 'number', min: 0 },
        { name: 'status', type: 'string', enum: ['open', 'in_progress', 'done', 'rejected'] },
        { name: 'authorId', type: 'string', required: true },
      ], [
        { fields: { category: 1, votes: -1 } },
        { fields: { status: 1 } },
      ]);
    } catch {
      // Collection already exists — ok
    }
  }

  return <FeedbackBoard />;
}
```

### 3. Create Records

```tsx
async function submitFeedback(db, userId) {
  const item = await db.create('feedback', {
    title: 'Dark mode support',
    body: 'Add dark mode toggle to the sidebar',
    category: 'feature',
    votes: 0,
    status: 'open',
    authorId: userId,
  });
  console.log('Created:', item._id);
  // item._createdAt and item._updatedAt are auto-set
}
```

### 4. Query with Filters

```tsx
async function loadFeedback(db) {
  // Top-voted open items
  const { records, total } = await db.query('feedback')
    .where('status', '==', 'open')
    .orderBy('votes', 'desc')
    .limit(20)
    .execute();

  console.log(`Showing ${records.length} of ${total} open items`);

  // Count by category
  const { result: byCat } = await db.aggregate(
    'feedback', 'count', undefined, undefined, 'category'
  );
  // byCat → [{ _id: 'bug', result: 5 }, { _id: 'feature', result: 12 }, ...]

  // Average votes for features
  const { result: avgVotes } = await db.aggregate(
    'feedback', 'avg', 'votes',
    [{ field: 'category', op: '==', value: 'feature' }]
  );
}
```

### 5. Update and Delete

```tsx
// Upvote
async function upvote(db, feedbackId) {
  const item = await db.get('feedback', feedbackId);
  await db.update('feedback', feedbackId, { votes: item.votes + 1 });
}

// Change status
async function markDone(db, feedbackId) {
  await db.update('feedback', feedbackId, { status: 'done' });
}

// Soft-delete (sets _deletedAt, excluded from future queries)
async function removeFeedback(db, feedbackId) {
  await db.delete('feedback', feedbackId);
}

// Bulk status change
async function closeAllRejected(db) {
  await db.updateMany('feedback',
    [{ field: 'status', op: '==', value: 'rejected' }],
    { status: 'done' }
  );
}
```

### 6. References Between Collections

```tsx
// Register a "comments" collection referencing "feedback"
await db.registerCollection('comments', [
  { name: 'body', type: 'string', required: true },
  { name: 'authorId', type: 'string', required: true },
  { name: 'feedback', type: 'reference', refCollection: 'feedback', onDelete: 'cascade' },
]);

// Create a comment with reference
const comment = await db.create('comments', {
  body: 'Great idea! +1',
  authorId: userId,
  feedback: { _ref: `feedback/${feedbackId}` },
});

// Populate — resolve reference to full document
const populated = await db.populate('comments', [comment._id], ['feedback']);
// populated[0].feedback → { _id: '...', title: 'Dark mode support', ... }
```

**Cascade behavior:** When a feedback item is deleted, all its comments are automatically soft-deleted too (because `onDelete: 'cascade'`).

### 7. Room-Scoped vs Global

```tsx
// GLOBAL (default) — same data in every room
// Good for: user profiles, product catalogs, app settings
await db.registerCollection('app_settings', [...]);

// ROOM-SCOPED — each room gets its own isolated data
// Good for: project tasks, team polls, channel-specific notes
await db.registerCollection('poll_votes', [
  { name: 'userId', type: 'string', required: true },
  { name: 'choice', type: 'string', required: true },
], undefined, 'room');
// In room "abc123": stored in app_{appId}_abc123_poll_votes
// In room "xyz789": stored in app_{appId}_xyz789_poll_votes
```

---

## mcpapp.db vs mcpapp.lists — When to Use What

| | `mcpapp.db.*` | `mcpapp.lists.*` |
|---|---|---|
| **Data model** | Schema-first, typed fields | Kanban board with custom fields |
| **Query** | Full query builder, aggregation | Simple filter by stage/field |
| **Relations** | Typed references, cascade delete | No cross-list references |
| **Scope** | Global or room | Room only |
| **UI** | Build your own | Built-in Kanban/Table views |
| **Best for** | CRM, inventory, forms, analytics | Task boards, pipelines, simple lists |
| **Max collections** | 20 per app | Unlimited lists |

**Rule of thumb:** If you need a Kanban board, use Lists. If you need structured data with queries and relationships, use DB.

---

## Error Codes

| Error | Cause | Fix |
|-------|-------|-----|
| `Invalid collection name` | Name doesn't match `^[a-zA-Z][a-zA-Z0-9_]{0,63}$` | Use only letters, numbers, underscores |
| `App has reached maximum collections` | 20 collection limit per app | Drop unused collections |
| `Collection already registered` | Duplicate registration | Catch error on init, or check with `getSchema` |
| `Unknown field` | Query references field not in schema | Check field names match schema |
| `Unsafe field name` | Field contains `$` or `.` | Use alphanumeric + underscore only |
| `Insufficient scope` | Missing required OAuth scope | Add scope to manifest |
| `Cannot delete: restrict rule` | Referencing records block deletion | Delete referencing records first, or use `set-null` |
| `Batch size exceeds limit` | createMany > 100 records | Split into batches of 100 |
| `Invalid record ID` | ID is not a valid 24-char hex string | Check the ID format |
| `Record not found` | ID doesn't exist or was soft-deleted | Verify record exists |

---

## Limits

| Constraint | Value |
|-----------|-------|
| Collections per app | 20 |
| Query result cap | 1,000 docs |
| Count cap | 10,000 |
| Batch create | 100 records |
| Query timeout | 10 seconds (`maxTimeMS`) |
| Populate fields per call | 5 |
| Populate records per call | 1,000 |
| Collection name length | 1–64 chars |
| Reference populate depth | 1 level (no recursive) |
