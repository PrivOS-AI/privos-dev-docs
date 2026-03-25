# App Platform — API Reference

## REST Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `mcp-apps.connect` | POST | Admin | Register direct app by server URL |
| `mcp-apps.register-relay` | POST | Admin | Register relay app, returns clientId+clientSecret+relayUrl |
| `mcp-apps.list` | GET | User | List all apps (includes relayOnline status for relay apps) |
| `mcp-apps.get` | GET | User | Get app details by ID |
| `mcp-apps.relay-status` | GET | User | Check if relay app is currently online |
| `mcp-apps.refresh` | POST | Admin | Re-fetch tools and metadata |
| `mcp-apps.delete` | POST | Admin | Remove app, revoke tokens |
| `mcp-apps.install` | POST | User | Install app in a room |
| `mcp-apps.uninstall` | POST | User | Remove from room |
| `mcp-apps.installations` | GET | User | List room installations |
| `mcp-apps.updateSettings` | POST | Admin | Update install perms, status, wakeUrl, queueTtl, queueMaxSize |
| `mcp-apps.tool-call` | POST | User | Execute a Privos MCP tool |
| `mcp-apps.ui-resource` | GET | User | Fetch UI HTML (server proxy) |
| `/apps/{appId}/ui` | GET | User | Per-app namespaced UI endpoint |
| `/apps/{appId}/mcp` | POST | User | Per-app namespaced JSON-RPC proxy |
| `wss://host/api/v1/mcp-apps.relay` | WebSocket | OAuth Bearer | Relay endpoint (token in Authorization header) |

## Privos MCP Tools

Tools apps can call via `callServerTool()` or React hooks:

| Tool | Scope | Description |
|------|-------|-------------|
| `privos.context.get` | — | Room ID, name, user roles |
| `privos.lists.getAll` | lists:read | All lists in room |
| `privos.lists.get` | lists:read | Single list by ID with field definitions |
| `privos.lists.getItems` | lists:read | Items in a list |
| `privos.lists.createItem` | lists:write | Create list item with custom field values |
| `privos.lists.updateItem` | lists:write | Update list item |
| `privos.lists.deleteItem` | lists:write | Delete list item |
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
| `name` | string | Yes | Field label |
| `type` | enum | Yes | `TEXT`, `TEXTAREA`, `NUMBER`, `DATE`, `DATE_TIME`, `SELECT`, `MULTI_SELECT`, `CHECKBOX`, `URL` |
| `options` | array | No | For SELECT/MULTI_SELECT: `[{ value: string }]` |

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

### Register Relay App

**POST `/api/v1/mcp-apps.register-relay`** (Admin only)

```json
{
  "manifestUrl": "https://myapp.example.com/.well-known/mcp/manifest.json",
  "connectionType": "relay"
}
```

**Response:**
```json
{
  "_id": "app_123",
  "clientId": "client_abc",
  "clientSecret": "secret_xyz",
  "relayUrl": "wss://chat.privos.com/api/v1/mcp-apps.relay"
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

## Scope Taxonomy

| Scope | Grants |
|-------|--------|
| `lists:read` | Read lists and items |
| `lists:write` | Create/update/delete lists, items, and fields |
| `files:read` | Read files and folders |
| `files:write` | Upload/delete files |
| `messages:read` | Read messages in authorized rooms |
| `messages:send` | Send messages in authorized rooms |
| `users:read` | Read user profiles |
| `rooms:read` | Read room metadata |
| `rooms:write` | Create/update rooms |
