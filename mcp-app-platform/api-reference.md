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

> **REST-first.** Lists, items, stages, files, folders, messages, rooms and users are
> reached through the hub REST API via `app.rest()` / `app.uploadFile()`, gated by the
> app's granted scopes — see [Auth & REST Integration](./auth-and-rest-integration.md).
> The MCP tools below remain only for capabilities with **no** REST equivalent.

Tools apps can call via `callServerTool()` or React hooks:

| Tool | Scope | Description |
|------|-------|-------------|
| `mcpapp.context.get` | — | User + room context: userId, username, signed userToken, appId, roomId/name/slug/type, appUrl, user roles, isAgentRoom flag, agent/default bot |
| `mcpapp.bot.joinCurrentRoom` | bot:room:join | Add the associated bot to the Hub-resolved current Room; accepts no Room or bot selector |
| `mcpapp.bot.getCurrentRoomIdentity` | bot:identity:read | Return safe associated-bot identity only with ordinary membership in the Hub-resolved current Room |
| `mcpapp.bot.getMe` | — | Validate a bot token, return bot identity |
| `mcpapp.bot.sendMessage` | bot:message:send | Send a text message to a room as a bot |
| `mcpapp.bot.editMessage` | bot:message:send | Edit the text or inline keyboard of a message the bot sent |
| `mcpapp.bot.deleteMessage` | bot:message:send | Delete a message the bot sent |
| `mcpapp.bot.sendDirectMessage` | bot:message:send | Send a DM to a user as a bot (shared-room required) |
| `mcpapp.bot.sendAttachment` | bot:message:send | Send photo/video/audio/document/voice to a room or DM (via fileId or fileUrl) |
| `mcpapp.notifications.create` | `notifications:write` | Notify one active member of the app's approved room through bell, native mobile, and Web Push |
| `mcpapp.app.getLocalData` | — | Get app's localData storage |
| `mcpapp.app.setLocalData` | — | Set/update app's localData |
| `mcpapp.app.deleteLocalData` | — | Delete key from localData |
| `mcpapp.app.clearLocalData` | — | Clear all localData (requires confirmation) |
| `mcpapp.db.registerCollection` | db:schema:write | Register app DB collection with schema |
| `mcpapp.db.updateSchema` | db:schema:write | Update collection schema fields |
| `mcpapp.db.getSchema` | db:schema:read | Get collection schema definition |
| `mcpapp.db.listCollections` | db:schema:read | List all app collections |
| `mcpapp.db.dropCollection` | db:schema:write | Drop collection and schema |
| `mcpapp.db.create` | db:write | Create record in collection |
| `mcpapp.db.createMany` | db:write | Batch create records (max 100) |
| `mcpapp.db.get` | db:read | Get record by ID |
| `mcpapp.db.update` | db:write | Update record by ID |
| `mcpapp.db.updateMany` | db:write | Update records matching filter |
| `mcpapp.db.delete` | db:write | Soft-delete record by ID |
| `mcpapp.db.deleteMany` | db:write | Soft-delete records matching filter |
| `mcpapp.db.query` | db:read | Query with filters, sort, pagination |
| `mcpapp.db.count` | db:read | Count records matching filter |
| `mcpapp.db.aggregate` | db:read | Aggregation (count/sum/avg/min/max) |
| `mcpapp.db.populate` | db:read | Resolve reference fields (1-level) |
| `mcpapp.objects.put` | db:write | Create or exactly adopt one immutable, content-addressed, room-private object |
| `mcpapp.objects.head` | db:read | Read metadata for one exact object (no content) |
| `mcpapp.objects.get` | db:read | Read one exact object's metadata + content, integrity-verified |

> **Database API:** See [apis/tools-database.md](./apis/tools-database.md) for full database and object-storage tool documentation with examples.
>
> **Data operations (lists, items, files, folders, messages, rooms, users):** call the hub
> REST API via `app.rest()` — see [Auth & REST Integration](./auth-and-rest-integration.md).

## Relay App Endpoints

### Generate Pairing URL

**POST `/api/v1/mcp-apps.generate-pair-url`** (Admin only)

Generates a one-time pairing URL for relay app developers to enter at `npm run pair`.

**Request:**
```json
{}
```

Send `{ "mcpAppId": "<id>" }` instead to pair an app that was already installed from its manifest.

**Response:**
```json
{
  "pairUrl": "wss://<hub>/api/v1/mcp-apps.relay?pair=<token>",
  "pairToken": "<token>",
  "fingerprint": "<hub fingerprint>"
}
```

### Check Pairing Status

**GET `/api/v1/mcp-apps.pair-status?token=<token>`** (Admin)

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
| `basic:information` | Read basic room/app identifiers: `roomId`, `roomSlug`, `appId`, and the app URL (returned by `mcpapp.context.get`) |
| `lists:read` | Read lists, items, stages, and field definitions |
| `lists:query` | Run filtered, paginated queries over list items (`POST items.query`). Separate from `lists:read` because it is a POST, and every `*:read` scope is GET-only |
| `lists:write` | Create/update/delete lists, items, stages, and fields |
| `files:read` | Read files and folders |
| `files:write` | Upload/update/delete files and folders |
| `messages:read` | Read messages in authorized rooms |
| `messages:send` | Send messages in authorized rooms |
| `notifications:write` | Notify one active member of the approved room through bell, native mobile, and Web Push |
| `users:read` | Read user profiles |
| `rooms:read` | Read room metadata and members |
| `rooms:write` | Create/update rooms |
| `bot:room:join` | Add the installation-owned bot (declared via the manifest `agentBot` block, created by a workspace admin through `mcp-apps.bot-agent.create`) to the exact active child Room binding (interactive user only) |
| `bot:identity:read` | Read safe associated-bot identity only for ordinary membership in the exact active child Room binding |
| `bot:message:send` | Send messages/DMs/attachments acting as a bot (runtime identity via bot token) |
| `db:schema:read` | List collections and read schema definitions |
| `db:schema:write` | Register/update/drop app DB collections and schemas |
| `db:read` | Read records, query, count, aggregate |
| `db:write` | Create/update/delete records (soft-delete) |
| `sandbox:generate` | Run a Sandbox agent generation (sync/async), upload files to attach, and — for an `operationId`-dispatched attempt — observe/cancel it and read its LLM evidence (see [Auth & REST Integration](./auth-and-rest-integration.md#idempotent-dispatch-with-operationid)) |
| `sandbox:skills:use` | List catalog (with per-room `selected`), read the room selection (`rooms.getPrivOSSandboxSelection`), and enable/disable skills + agent sets for a room — replace or merge mode (room admin) |
| `sandbox:botkey:push` | Provision the room's Sandbox project + push/refresh its bot key (room admin) |
| `sandbox:wake` | Wake/establish a room's Sandbox VM without re-pushing the bot key (`agents.sandbox.wake`) + poll its state (`agents.sandbox.vmState`) (room admin) |
| `sandbox:ai-chat` | List a room's AI Chat sessions and read their message history (`ai-messages.sessions` / `.getSession` / `.list`) |
| `sandbox:ai-chat:write` | Send messages in a room's AI Chat, start the agent generation, and cancel an in-flight run (`ai-messages.send` / `.startGeneration` / `.cancel`) |
| `sandbox:agent-sets:upload` | Manage the workspace Agent Factory catalog: submit agent-set archives for an admin to confirm (`agents.sandbox.agentSets.preview` / `.confirm`) and remove a set (`agents.sandbox.agentSets.delete`, refused while any room still references it). Workspace-context, interactive user only, risk `critical`; the endpoints additionally enforce the native `manage-privos-agent-sets` permission, so the scope only widens reachable paths |

### Tools that do NOT require a scope

The following tools bypass the OAuth scope check (they only need the app to be installed and the user to have access to the room):

| Tool | Reason |
|------|--------|
| `mcpapp.context.get` | Pure read of current user/room context — no resource access |
| `mcpapp.bot.getMe` | Validates a bot token (runtime identity check, not resource access) |
| `mcpapp.app.getLocalData` / `setLocalData` / `deleteLocalData` / `clearLocalData` | App's own sandboxed `localData` store — scoped to the app itself |
