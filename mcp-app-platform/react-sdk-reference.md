# App Platform — React SDK Reference

Package: `@privos/app-react`

Thin React wrapper around the MCP `@modelcontextprotocol/ext-apps` SDK with PrivOS-specific convenience hooks.

## Hooks

| Hook | Returns | Description |
|------|---------|-------------|
| `usePrivOSApp()` | `McpApp` | MCP app instance for direct `callServerTool()` |
| `usePrivOSContext()` | `PrivOSContext` | userId, roomId, theme, userRoles, isAgentRoom, agent/default bot |
| `usePrivOSTool(name, args)` | `{ data, loading, error, refetch }` | Generic auto-fetching tool call |
| `useLists(roomId)` | `{ data, loading, error }` | Lists in room |
| `useFiles(roomId)` | `{ data, loading, error }` | Files in room |
| `useRoom(roomId?)` | `{ data, loading, error }` | Room metadata |

## Provider

Wrap your app with `PrivOSAppProvider`:

```tsx
import { PrivOSAppProvider } from '@privos/app-react';

export default function App() {
  return (
    <PrivOSAppProvider>
      <MyComponent />
    </PrivOSAppProvider>
  );
}
```

## usePrivOSApp

Returns the MCP app instance. Use for mutations (writes) where you call tools directly:

```tsx
const app = usePrivOSApp();

await app.callServerTool({
  name: 'privos.lists.createItem',
  arguments: { listId: 'abc', title: 'New item' }
});
```

### app.rest() — call hub REST endpoints

Preferred over resource tools for accessing existing hub data. The host proxies
the call as the current user; the server gates it against the app's granted
scopes (a request to a path the app's scopes don't cover is rejected with 403).

```tsx
const app = usePrivOSApp();

// GET — list files in a channel
const files = await app.rest({
  method: 'GET',
  path: 'file-management.files.channel/' + roomId,
  query: { count: 50 },
});

// POST — JSON body
await app.rest({
  method: 'POST',
  path: 'file-management.folders.create',
  body: { channelId: roomId, name: 'Reports' },
});
```

`RestRequestParams`:

| Field | Type | Notes |
|-------|------|-------|
| `method` | `'GET' \| 'POST' \| 'PUT' \| 'PATCH' \| 'DELETE'` | required |
| `path` | `string` | hub REST path after `/api/v1/` |
| `query` | `Record<string, string \| number \| boolean>` | optional querystring |
| `body` | `any` | optional JSON body |

### app.uploadFile() — multipart file upload

Uploads a file to file management as the current user. Requires the
**`files:write`** scope in the app manifest (admin-granted).

```tsx
const app = usePrivOSApp();

await app.uploadFile({
  channelId: roomId,
  fileName: 'report.pdf',
  base64Data,                  // base64 string or data URI
  mimeType: 'application/pdf',
  // folderId, enableEmbedding, duplicateAction are optional
});
```

`UploadFileParams`:

| Field | Type | Notes |
|-------|------|-------|
| `channelId` | `string` | required — target room |
| `fileName` | `string` | required |
| `base64Data` | `string` | required — base64 content (data URI accepted) |
| `mimeType` | `string` | optional |
| `folderId` | `string` | optional |
| `enableEmbedding` | `boolean` | optional |
| `duplicateAction` | `'replace' \| 'keep_both' \| 'cancel'` | optional |

> The app never exceeds the current user's own permissions — the server still
> enforces per-room ACL and read-only restrictions on every call.

## usePrivOSContext

Subscribes to `HOST_CONTEXT_CHANGED` push + fetches PrivOS-specific context:

```tsx
const {
  roomId, userId, username, roomName, roomType, theme, userRoles,
  isAgentRoom, agentBot, defaultBot,
} = usePrivOSContext();
```

- `theme` (`'light'` | `'dark'`) — updates in real-time when the user toggles theme
- `isAgentRoom` (`boolean`) — `true` when the room was created for an agent
- `agentBot` (`{ _id, username } | null`) — present when `isAgentRoom` is `true`; `null` if bot left the room
- `defaultBot` (`{ _id, username } | null`) — present when `isAgentRoom` is `false`; the room's auto-provisioned bot, or `null` if removed

Use with a `ThemeProvider` for Auto/Light/Dark mode support. See [Developer Guide — Theme Sync](./developer-guide.md#7-theme-sync-lightdark-mode).

## usePrivOSTool

Generic hook — auto-fetches on mount and when args change. Best for reads:

```tsx
const { data, loading, error, refetch } = usePrivOSTool('privos.lists.get', { listId });
```

**Note:** Skips fetch if any arg value is empty/null/undefined.

## useLists

Convenience wrapper for `privos.lists.getAll`:

```tsx
const { data: lists, loading, error } = useLists(roomId);
```

## useAppDb

Database operations hook — provides typed CRUD, query builder, and schema management:

```tsx
import { useAppDb } from '@privos/app-react';

function MyComponent() {
  const db = useAppDb();

  // Schema management
  await db.registerCollection('contacts', [
    { name: 'name', type: 'string', required: true },
    { name: 'email', type: 'string' },
  ]);

  // CRUD
  const contact = await db.create('contacts', { name: 'Alice' });
  const found = await db.get('contacts', contact._id);
  await db.update('contacts', contact._id, { name: 'Bob' });
  await db.delete('contacts', contact._id);

  // Query builder (chainable)
  const results = await db.query('contacts')
    .where('name', '!=', '')
    .orderBy('name', 'asc')
    .limit(10)
    .execute();

  // Aggregation
  const stats = await db.aggregate('contacts', 'count');
}
```

### Methods

| Method | Scope | Description |
|--------|-------|-------------|
| `registerCollection(col, fields, indexes?)` | db:schema:write | Register collection with schema |
| `updateSchema(col, fields)` | db:schema:write | Update schema fields |
| `getSchema(col)` | db:schema:read | Get schema definition |
| `listCollections()` | db:schema:read | List all app collections |
| `dropCollection(col)` | db:schema:write | Drop collection |
| `create(col, data)` | db:write | Create record |
| `createMany(col, records)` | db:write | Batch create (max 100) |
| `get(col, id)` | db:read | Get record by ID |
| `update(col, id, data)` | db:write | Update record |
| `delete(col, id)` | db:write | Soft-delete record |
| `query(col)` | db:read | Returns `QueryBuilder` |
| `count(col, where?)` | db:read | Count records |
| `aggregate(col, op, field?, where?, groupBy?)` | db:read | Aggregation |
| `populate(col, ids, fields)` | db:read | Resolve references |

### QueryBuilder

Chainable query builder returned by `db.query(collection)`:

```tsx
db.query('contacts')
  .where('status', '==', 'active')
  .where('age', '>=', 18)
  .orderBy('name', 'asc')
  .limit(20)
  .offset(0)
  .populate('company')
  .execute()  // → { records, total }
  .count()    // → { count }
```

## Pattern: Reads vs Mutations

- **Reads** — use `usePrivOSTool` or convenience hooks (`useLists`, `useFiles`, `useRoom`). Auto-fetches.
- **Hub data (REST-first)** — use `app.rest()` to reach existing hub REST endpoints (files, folders, rooms…) as the current user, gated by the app's scopes. Preferred over building dedicated resource tools.
- **File upload** — use `app.uploadFile()` (requires `files:write`).
- **Mutations** — use `usePrivOSApp()` to get the app instance, then call `app.callServerTool()` directly in event handlers.
- **Database** — use `useAppDb()` for all database operations (both reads and writes).
