# REST API — App Settings

## Update Settings

Update an app's configuration including install permissions, status, and relay settings.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/mcp-apps.updateSettings` |
| **Auth** | Admin (`manage-oauth-apps` permission) |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `_id` | string | Yes | App ID |
| `installPermission` | string[] | No | Roles allowed to install. Values: `admin`, `owner`, `leader`. At least one required. |
| `status` | string | No | App status: `draft`, `active`, `suspended` |
| `uiUrl` | string | No | UI resource URL |
| `wakeUrl` | string | No | URL to wake a sleeping relay app. Must start with `https://` |
| `queueTtlMs` | number | No | Queue message TTL in milliseconds. Clamped to 1000-120000 |
| `queueMaxSize` | number | No | Maximum queue size. Clamped to 1-100 |

### Example: Update Install Permissions

```json
{
  "_id": "app_abc123",
  "installPermission": ["admin", "owner"]
}
```

### Example: Change Status

```json
{
  "_id": "app_abc123",
  "status": "suspended"
}
```

### Example: Configure Relay Queue

```json
{
  "_id": "app_abc123",
  "wakeUrl": "https://myapp.example.com/wake",
  "queueTtlMs": 30000,
  "queueMaxSize": 50
}
```

### Response

```json
{
  "success": true,
  "app": {
    "_id": "app_abc123",
    "name": "My MCP App",
    "status": "active",
    "installPermission": ["admin", "owner"],
    "wakeUrl": "https://myapp.example.com/wake",
    "queueTtlMs": 30000,
    "queueMaxSize": 50,
    "updatedAt": "2026-03-25T15:00:00Z"
  }
}
```

### Validation Rules

| Field | Rule |
|-------|------|
| `installPermission` | Must be array of `admin`, `owner`, or `leader`. At least one value required. |
| `status` | Must be `draft`, `active`, or `suspended` |
| `wakeUrl` | Must start with `https://` (SSRF prevention) |
| `queueTtlMs` | Automatically clamped to range 1000-120000 ms |
| `queueMaxSize` | Automatically clamped to range 1-100 |

### Errors

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `_id is required` | Missing app ID |
| 400 | `Invalid installPermission` | Invalid role values |
| 400 | `Invalid status` | Status not one of allowed values |
| 400 | `wakeUrl must use HTTPS` | Non-HTTPS wake URL |
| 403 | `Not authorized` | User lacks permission |
| 404 | `App not found` | No app with this ID |
