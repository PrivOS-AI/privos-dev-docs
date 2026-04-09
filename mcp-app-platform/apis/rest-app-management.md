# REST API — App Management

## Connect App

Register a direct MCP app by providing its server URL.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/mcp-apps.connect` |
| **Auth** | Admin (`manage-oauth-apps` permission) |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `serverUrl` | string | Yes | The MCP server URL to connect to |

```json
{
  "serverUrl": "https://myapp.example.com/mcp"
}
```

### Response

```json
{
  "success": true,
  "app": {
    "_id": "app_abc123",
    "name": "My MCP App",
    "description": "An example app",
    "serverUrl": "https://myapp.example.com/mcp",
    "type": "direct",
    "status": "active",
    "tools": [
      {
        "name": "myTool",
        "description": "Does something useful",
        "inputSchema": { "type": "object", "properties": {} }
      }
    ],
    "scopes": ["lists:read", "lists:write"],
    "icon": "icon.png",
    "createdAt": "2026-03-25T10:00:00Z",
    "updatedAt": "2026-03-25T10:00:00Z"
  },
  "clientId": "client_xyz",
  "clientSecret": "secret_abc"
}
```

### Errors

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `serverUrl is required` | Missing server URL |
| 403 | `Not authorized` | User lacks `manage-oauth-apps` permission |
| 500 | `Failed to connect` | Cannot reach the MCP server |

---

## List Apps

List all registered MCP apps. Relay apps include `relayOnline` status.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/mcp-apps.list` |
| **Auth** | User |

### Request

No parameters required.

### Response

```json
{
  "success": true,
  "apps": [
    {
      "_id": "app_abc123",
      "name": "My Direct App",
      "type": "direct",
      "status": "active",
      "serverUrl": "https://myapp.example.com/mcp",
      "tools": [...],
      "scopes": ["lists:read"],
      "icon": "icon.png",
      "createdAt": "2026-03-25T10:00:00Z",
      "updatedAt": "2026-03-25T10:00:00Z"
    },
    {
      "_id": "app_def456",
      "name": "My Relay App",
      "type": "relay",
      "status": "active",
      "relayOnline": true,
      "tools": [...],
      "scopes": ["messages:read", "messages:send"],
      "icon": "icon.png",
      "createdAt": "2026-03-25T11:00:00Z",
      "updatedAt": "2026-03-25T11:00:00Z"
    }
  ]
}
```

---

## Get App

Get details of a specific app by ID.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/mcp-apps.get` |
| **Auth** | User |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `_id` | string (query) | Yes | App ID |

```
GET /api/v1/mcp-apps.get?_id=app_abc123
```

### Response

```json
{
  "success": true,
  "app": {
    "_id": "app_abc123",
    "name": "My MCP App",
    "description": "An example app",
    "type": "direct",
    "status": "active",
    "serverUrl": "https://myapp.example.com/mcp",
    "tools": [
      {
        "name": "myTool",
        "description": "Does something useful",
        "inputSchema": { "type": "object", "properties": {} }
      }
    ],
    "scopes": ["lists:read", "lists:write"],
    "icon": "icon.png",
    "installPermission": ["admin", "owner"],
    "relayOnline": true,
    "createdAt": "2026-03-25T10:00:00Z",
    "updatedAt": "2026-03-25T10:00:00Z"
  }
}
```

### Errors

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `_id is required` | Missing app ID |
| 404 | `App not found` | No app with this ID |

---

## Refresh App

Re-fetch tools and metadata from the MCP server.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/mcp-apps.refresh` |
| **Auth** | Admin (`manage-oauth-apps` permission) |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `_id` | string | Yes | App ID to refresh |

```json
{
  "_id": "app_abc123"
}
```

### Response

```json
{
  "success": true,
  "app": {
    "_id": "app_abc123",
    "name": "My MCP App",
    "tools": [...],
    "updatedAt": "2026-03-25T15:00:00Z"
  }
}
```

### Errors

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `_id is required` | Missing app ID |
| 403 | `Not authorized` | User lacks permission |
| 404 | `App not found` | No app with this ID |

---

## Delete App

Remove an app and revoke all associated tokens and installations.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/mcp-apps.delete` |
| **Auth** | Admin (`manage-oauth-apps` permission) |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `_id` | string | Yes | App ID to delete |

```json
{
  "_id": "app_abc123"
}
```

### Response

```json
{
  "success": true
}
```

### Errors

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `_id is required` | Missing app ID |
| 403 | `Not authorized` | User lacks permission |
| 404 | `App not found` | No app with this ID |

---

## Data Model: `IMcpApp`

| Field | Type | Description |
|-------|------|-------------|
| `_id` | string | Unique app identifier |
| `name` | string | App display name |
| `description` | string | App description |
| `type` | `"direct"` \| `"relay"` | Connection type |
| `status` | `"draft"` \| `"active"` \| `"suspended"` | App status |
| `serverUrl` | string | MCP server URL (direct apps) |
| `tools` | array | Array of MCP tool definitions |
| `scopes` | string[] | Granted permission scopes |
| `icon` | string | Icon filename (served via file management API) |
| `installPermission` | string[] | Roles allowed to install: `admin`, `owner`, `leader` |
| `uiUrl` | string | UI resource URL |
| `wakeUrl` | string | HTTPS URL to wake relay app |
| `queueTtlMs` | number | Queue TTL in ms (1000-120000) |
| `queueMaxSize` | number | Max queue size (1-100) |
| `relayOnline` | boolean | Whether relay app is currently connected (relay apps only) |
| `createdAt` | Date | Creation timestamp |
| `updatedAt` | Date | Last update timestamp |
