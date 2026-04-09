# REST API — Installations

## Install App

Install an MCP app into a room.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/mcp-apps.install` |
| **Auth** | User (must have role listed in app's `installPermission`) |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mcpAppId` | string | Yes | App ID to install |
| `roomId` | string | No | Target room ID |

```json
{
  "mcpAppId": "app_abc123",
  "roomId": "room_xyz789"
}
```

### Response

```json
{
  "success": true,
  "installation": {
    "_id": "inst_001",
    "mcpAppId": "app_abc123",
    "roomId": "room_xyz789",
    "installedBy": "user_123",
    "createdAt": "2026-03-25T10:00:00Z"
  }
}
```

### Errors

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `mcpAppId is required` | Missing app ID |
| 403 | `Not authorized to install` | User's role not in `installPermission` |
| 404 | `App not found` | No app with this ID |

---

## Uninstall App

Remove an app installation from a room.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/mcp-apps.uninstall` |
| **Auth** | User |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `installationId` | string | Yes | Installation ID to remove |

```json
{
  "installationId": "inst_001"
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
| 400 | `installationId is required` | Missing installation ID |
| 404 | `Installation not found` | No installation with this ID |

---

## List Room Installations

List all app installations in a specific room.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/mcp-apps.installations` |
| **Auth** | User |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `roomId` | string (query) | Yes | Room ID to list installations for |

```
GET /api/v1/mcp-apps.installations?roomId=room_xyz789
```

### Response

```json
{
  "success": true,
  "installations": [
    {
      "_id": "inst_001",
      "mcpAppId": "app_abc123",
      "roomId": "room_xyz789",
      "installedBy": "user_123",
      "createdAt": "2026-03-25T10:00:00Z"
    },
    {
      "_id": "inst_002",
      "mcpAppId": "app_def456",
      "roomId": "room_xyz789",
      "installedBy": "user_456",
      "createdAt": "2026-03-26T08:00:00Z"
    }
  ]
}
```

---

## Data Model: `IMcpAppInstallation`

| Field | Type | Description |
|-------|------|-------------|
| `_id` | string | Unique installation identifier |
| `mcpAppId` | string | Reference to the MCP app |
| `roomId` | string | Room where the app is installed |
| `installedBy` | string | User ID who installed the app |
| `createdAt` | Date | Installation timestamp |
