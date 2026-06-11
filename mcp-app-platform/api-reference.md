# App Platform — API Reference

> **Detailed documentation available:** See [apis/](./apis/) for comprehensive API docs with full request/response examples, data models, and error codes.

## REST Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `mcp-apps.connect` | POST | Admin | Register direct app by server URL |
| `mcp-apps.generate-pair-url` | POST | Admin | Generate one-time pairing URL for relay apps, returns { pairUrl, pairToken } |
| `mcp-apps.pair-status` | GET | Admin | Check pairing status by token, returns { status: 'waiting'\|'paired'\|'expired', app? } |
| `mcp-apps.list` | GET | User | List all apps (includes relayOnline status for relay apps) |
| `mcp-apps.get` | GET | User | Get app details by ID |
| `mcp-apps.relay-status` | GET | User | Check if relay app is currently online |
| `mcp-apps.refresh` | POST | Admin | Re-fetch tools and metadata |
| `mcp-apps.delete` | POST | Admin | Remove app, revoke tokens |
| `mcp-apps.install` | POST | User | Install app in a room |
| `mcp-apps.uninstall` | POST | User | Remove from room |
| `mcp-apps.installations` | GET | User | List room installations |
| `mcp-apps.updateSettings` | POST | Admin | Update install perms, status, wakeUrl, queueTtl, queueMaxSize |
| `mcp-apps.tool-call` | POST | User | Execute a PrivOS MCP tool |
| `mcp-apps.ui-resource` | GET | User | Fetch UI HTML (server proxy) |
| `/apps/{appId}/ui` | GET | User | Per-app namespaced UI endpoint |
| `/apps/{appId}/mcp` | POST | User | Per-app namespaced JSON-RPC proxy |
| `wss://host/api/v1/mcp-apps.relay` | WebSocket | OAuth Bearer | Relay endpoint (token in Authorization header) |

## PrivOS MCP Tools

> **Resource tools below are legacy.** For files, folders, lists, stages, messages, rooms
> and users, prefer the hub REST API via `app.rest()` / `app.uploadFile()` (gated by the
> app's granted scopes). See [Auth & REST Integration](./auth-and-rest-integration.md).
> Tools with **no** REST equivalent stay current: `privos.context.get`, `privos.app.*LocalData`,
> `privos.db.*`, `privos.bot.getMe`.

Tools apps can call via `callServerTool()` or React hooks:

| Tool | Scope | Description |
|------|-------|-------------|
| `privos.context.get` | — | Room ID/name/type, user roles, isAgentRoom flag, agent/default bot |
| `privos.bot.getMe` | — | Validate a bot token, return bot identity |
| `privos.bot.sendMessage` | bot:message:send | Send a text message to a room as a bot |
| `privos.bot.sendDirectMessage` | bot:message:send | Send a DM to a user as a bot (shared-room required) |
| `privos.bot.sendAttachment` | bot:message:send | Send photo/video/audio/document/voice to a room or DM (via fileId or fileUrl) |
| `privos.users.getByIds` | users:read | Batch lookup of public profiles by `userIds` or `usernames` (one wins; max 100) |
| `privos.app.getLocalData` | — | Get app's localData storage |
| `privos.app.setLocalData` | — | Set/update app's localData |
| `privos.app.deleteLocalData` | — | Delete key from localData |
| `privos.app.clearLocalData` | — | Clear all localData (requires confirmation) |
| `privos.lists.create` | lists:write | Create a new list with fields and stages |
| `privos.lists.getAll` | lists:read | All lists in room |
| `privos.lists.get` | lists:read | Single list by ID or key, with field definitions |
| `privos.lists.getItems` | lists:read | Items in a list (offset/count pagination, sortable, filterable by stage) |
| `privos.lists.createItem` | lists:write | Create list item with custom field values |
| `privos.lists.updateItem` | lists:write | Update list item |
| `privos.lists.deleteItem` | lists:write | Delete list item |
| `privos.lists.deleteItems` | lists:write | Batch delete items by IDs or custom field filter |
| `privos.lists.addField` | lists:write | Add custom field definition to list |
| `privos.lists.removeField` | lists:write | Remove custom field definition from list |
| `privos.lists.updateList` | lists:write | Update list name/description |
| `privos.lists.getItem` | lists:read | Full item detail by ID (all custom fields) |
| `privos.lists.getItemsByStage` | lists:read | Items in a specific stage |
| `privos.lists.moveItemToStage` | lists:write | Move item to different stage (kanban) |
| `privos.lists.reorderItem` | lists:write | Change item position within stage |
| `privos.lists.updateCustomField` | lists:write | Update a single custom field on an item |
| `privos.lists.searchItems` | lists:read | Search items by name |
| `privos.lists.getSubItems` | lists:read | Get child/sub-items of a parent item |
| `privos.lists.getItemsByStages` | lists:read | Get items grouped by multiple stages |
| `privos.lists.batchAddFields` | lists:write | Add multiple field definitions in one call |
| `privos.lists.batchCreateItems` | lists:write | Create multiple items in one call |
| `privos.stages.getByList` | lists:read | Get all stages (kanban columns) for a list |
| `privos.stages.get` | lists:read | Get single stage by ID |
| `privos.stages.create` | lists:write | Create a new stage |
| `privos.stages.update` | lists:write | Update stage name/color |
| `privos.stages.delete` | lists:write | Delete a stage |
| `privos.stages.reorder` | lists:write | Reorder stages by providing ordered IDs |
| `privos.files.getByChannel` | files:read | Files in room, optionally by folder |
| `privos.files.get` | files:read | File detail by ID |
| `privos.files.search` | files:read | Search files by name |
| `privos.files.count` | files:read | Count files in channel/folder |
| `privos.files.update` | files:write | Update file name/description/folder |
| `privos.files.delete` | files:write | Delete file |
| `privos.folders.getByChannel` | files:read | Folders in channel, optionally by parent |
| `privos.folders.get` | files:read | Folder detail by ID |
| `privos.folders.getContent` | files:read | Files and subfolders inside a folder |
| `privos.folders.getRootContent` | files:read | Root-level files and folders |
| `privos.folders.create` | files:write | Create folder |
| `privos.folders.update` | files:write | Rename/move folder |
| `privos.folders.delete` | files:write | Delete folder (optionally recursive) |
| `privos.folders.search` | files:read | Search folders by name |
| `privos.messages.getRecent` | messages:read | Recent room messages |
| `privos.messages.send` | messages:send | Send message to room |
| `privos.rooms.get` | rooms:read | Room metadata |
| `privos.rooms.getMembers` | rooms:read | Room member list |
| `privos.users.get` | users:read | User profile by ID |
| `privos.users.getCurrent` | users:read | Current user profile |
| `privos.db.registerCollection` | db:schema:write | Register app DB collection with schema |
| `privos.db.updateSchema` | db:schema:write | Update collection schema fields |
| `privos.db.getSchema` | db:schema:read | Get collection schema definition |
| `privos.db.listCollections` | db:schema:read | List all app collections |
| `privos.db.dropCollection` | db:schema:write | Drop collection and schema |
| `privos.db.create` | db:write | Create record in collection |
| `privos.db.createMany` | db:write | Batch create records (max 100) |
| `privos.db.get` | db:read | Get record by ID |
| `privos.db.update` | db:write | Update record by ID |
| `privos.db.updateMany` | db:write | Update records matching filter |
| `privos.db.delete` | db:write | Soft-delete record by ID |
| `privos.db.deleteMany` | db:write | Soft-delete records matching filter |
| `privos.db.query` | db:read | Query with filters, sort, pagination |
| `privos.db.count` | db:read | Count records matching filter |
| `privos.db.aggregate` | db:read | Aggregation (count/sum/avg/min/max) |
| `privos.db.populate` | db:read | Resolve reference fields (1-level) |

> **Database API:** See [apis/tools-database.md](./apis/tools-database.md) for full database tool documentation with examples.

## Tool Details

### `privos.lists.createItem`

Create a new item in a list with optional custom field values.

**Arguments:**

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `listId` | string | Yes | List ID |
| `title` | string | Yes | Item title/name |
| `description` | string | No | Item description |
| `customFields` | array | No | Array of `{ fieldId: string, value: any }` |

**Response:**
```json
{ "_id": "item_123", "name": "Item Title", "listId": "list_456" }
```

**Example:**
```typescript
await app.callServerTool({
  name: 'privos.lists.createItem',
  arguments: {
    listId: 'list_123',
    title: 'John Doe',
    customFields: [
      { fieldId: 'field_email', value: 'john@example.com' },
      { fieldId: 'field_phone', value: '+1-555-0100' },
    ]
  }
});
```

### `privos.lists.addField`

Add a new custom field definition to a list.

**Arguments:**

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `listId` | string | Yes | List ID |
| `fieldId` | string | No | Custom field ID (auto-generated if omitted) |
| `name` | string | Yes | Field label |
| `type` | enum | Yes | `TEXT`, `TEXTAREA`, `NUMBER`, `DATE`, `DATE_TIME`, `SELECT`, `MULTI_SELECT`, `CHECKBOX`, `URL`, `USER`, `FILE`, `FILE_MULTIPLE`, `DOCUMENT`, `ASSIGNEE`, `DEADLINE`, `DEPENDENCIES` |
| `options` | array | No | For SELECT/MULTI_SELECT: `[{ _id?, value, color? }]` |

**Response:**
```json
{ "_id": "field_789", "name": "Email", "type": "TEXT", "order": 3 }
```

**Example (SELECT with options):**
```typescript
await app.callServerTool({
  name: 'privos.lists.addField',
  arguments: {
    listId: 'list_123',
    name: 'Source',
    type: 'SELECT',
    options: [{ value: 'Web' }, { value: 'Email' }, { value: 'Phone' }]
  }
});
```

## Relay App Endpoints

### Generate Pairing URL

**POST `/api/v1/mcp-apps.generate-pair-url`** (Admin only)

Generates a one-time pairing URL for relay app developers to use during `npm start`.

**Request:**
```json
{
  "manifestUrl": "https://myapp.example.com/.well-known/mcp/manifest.json"
}
```

**Response:**
```json
{
  "pairUrl": "https://chat.privos.com/pair?token=eyJ...",
  "pairToken": "pair_abc_123xyz",
  "expiresIn": 3600
}
```

### Check Pairing Status

**GET `/api/v1/mcp-apps.pair-status?token=pair_abc_123xyz`** (Admin)

Check pairing progress during relay setup flow.

**Response (waiting):**
```json
{
  "status": "waiting",
  "createdAt": "2026-03-25T10:00:00Z",
  "expiresAt": "2026-03-25T11:00:00Z"
}
```

**Response (paired):**
```json
{
  "status": "paired",
  "app": {
    "_id": "app_123",
    "clientId": "client_abc",
    "clientSecret": "secret_xyz",
    "relayUrl": "wss://chat.privos.com/api/v1/mcp-apps.relay"
  }
}
```

### Check Relay Status

**GET `/api/v1/mcp-apps.relay-status?appId=app_123`** (User)

**Response:**
```json
{ "isOnline": true, "lastSeen": "2026-03-25T14:30:00Z" }
```

### OAuth Token (for relay apps)

**POST `/oauth/token`** (Client Credentials)

```bash
curl -X POST http://localhost:3000/oauth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET"
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

## Credential Push (Outbound from Hub)

After a successful `mcp-apps.connect` (direct apps only), the Hub does a **best-effort POST** to the MCP app server so it can auto-configure callback credentials without manual copy/paste.

### Endpoint (implemented by the app server, optional)

```
POST {serverUrl}/.well-known/mcp/register
Content-Type: application/json

{
  "appId": "my-app",
  "clientId": "client_abc",
  "clientSecret": "secret_xyz",
  "timestamp": "2026-05-25T10:30:00.000Z"
}
```

### Hub semantics

| Server response | Hub behavior |
|---|---|
| 2xx | `success` — credentials delivered |
| 404 | `skipped` — endpoint not implemented (no error logged) |
| any other status / timeout / network error | `failed` — warning logged, connect still succeeds |

- Timeout: 10s
- **Failure does NOT abort connect** — admin can still copy credentials manually from the connect response
- App developers are encouraged to implement this endpoint to enable zero-config callbacks (especially for direct apps that need OAuth tokens for `privos.*` callback tools)

### Per-App Namespaced Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/apps/{appId}/ui` | GET | Returns the app's UI HTML proxy (set by `uiUrl`). Cookies are isolated from other apps |
| `/apps/{appId}/mcp` | POST | JSON-RPC proxy — direct apps: forwards to `{serverUrl}/mcp`; relay apps: dispatched over the WS connection |

These give each app a per-app URL namespace so cookies/storage in iframes are partitioned.

## App Asset Storage

### File Management via MinIO

App icons and assets are stored in MinIO at path `.apps/{appId}/`.

**Serving Files:**
```
GET /api/v1/file-management.files/{fileId}/content/{filename}
```

**Icon Storage:**
- Icon stored as relative path in `mcp_apps.icon` field (e.g., `"icon.png"`)
- No domain prefix — served via file management API
- Accessed via: `GET /api/v1/file-management.files/{appIconFileId}/content/icon.png`
- Icon also included in pairing metadata and initialize response `serverInfo.icon`

## Scope Taxonomy

| Scope | Grants |
|-------|--------|
| `lists:read` | Read lists, items, stages, and field definitions |
| `lists:write` | Create/update/delete lists, items, stages, and fields |
| `files:read` | Read files and folders |
| `files:write` | Upload/update/delete files and folders |
| `messages:read` | Read messages in authorized rooms |
| `messages:send` | Send messages in authorized rooms |
| `users:read` | Read user profiles |
| `rooms:read` | Read room metadata and members |
| `rooms:write` | Create/update rooms |
| `bot:message:send` | Send messages/DMs/attachments acting as a bot (runtime identity via bot token) |
| `db:schema:read` | List collections and read schema definitions |
| `db:schema:write` | Register/update/drop app DB collections and schemas |
| `db:read` | Read records, query, count, aggregate |
| `db:write` | Create/update/delete records (soft-delete) |

### Tools that do NOT require a scope

The following tools bypass the OAuth scope check (they only need the app to be installed and the user to have access to the room):

| Tool | Reason |
|------|--------|
| `privos.context.get` | Pure read of current user/room context — no resource access |
| `privos.bot.getMe` | Validates a bot token (runtime identity check, not resource access) |
| `privos.app.getLocalData` / `setLocalData` / `deleteLocalData` / `clearLocalData` | App's own sandboxed `localData` store — scoped to the app itself |
