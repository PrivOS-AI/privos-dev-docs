# REST API — UI Resources

## Fetch UI Resource

Server-side proxy that fetches UI HTML content from an MCP app. Supports both direct HTTP apps and relay apps.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/mcp-apps.ui-resource` |
| **Auth** | User |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `resourceUri` | string (query) | Yes | URI of the UI resource to fetch |
| `mcpAppId` | string (query) | Conditional | App ID (required for relay apps) |
| `serverUrl` | string (query) | Conditional | Server URL (alternative to mcpAppId for direct apps) |

Either `mcpAppId` or `serverUrl` must be provided.

### Example: Fetch via App ID

```
GET /api/v1/mcp-apps.ui-resource?mcpAppId=app_abc123&resourceUri=panel://main
```

### Example: Fetch via Server URL

```
GET /api/v1/mcp-apps.ui-resource?serverUrl=https://myapp.example.com/mcp&resourceUri=panel://main
```

### Response

```json
{
  "success": true,
  "html": "<div class=\"my-app\"><h1>App Panel</h1>...</div>"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `html` | string | HTML content of the UI resource |

### Errors

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `resourceUri is required` | Missing resource URI |
| 400 | `mcpAppId or serverUrl is required` | Neither identifier provided |
| 404 | `App not found` | No app with this ID |
| 500 | `Failed to fetch resource` | Error fetching from MCP server |

---

## Per-App UI Endpoint

Each app also has a namespaced UI endpoint.

| | |
|---|---|
| **Endpoint** | `GET /apps/{appId}/ui` |
| **Auth** | User |

This endpoint serves the app's UI directly, using the app's configured `uiUrl`.

---

## Per-App MCP Endpoint

Each app has a namespaced JSON-RPC proxy endpoint.

| | |
|---|---|
| **Endpoint** | `POST /apps/{appId}/mcp` |
| **Auth** | User |

Proxies JSON-RPC requests to the MCP app server or relay connection.
