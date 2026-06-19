# REST API — Tool Execution

## Execute Tool Call

Execute an MCP tool on a specific app. Use this for tools with **no REST equivalent**
(`privos.db.*`, `privos.bot.*`, `privos.context.get`, `privos.app.*`). For lists, items,
files, folders, messages, rooms and users, call the hub REST API via `app.rest()` instead
— see [Auth & REST Integration](../auth-and-rest-integration.md).

| | |
|---|---|
| **Endpoint** | `POST /api/v1/mcp-apps.tool-call` |
| **Auth** | User |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mcpAppId` | string | Yes | App ID to execute the tool on |
| `toolName` | string | Yes | Name of the tool to execute |
| `arguments` | object | No | Tool-specific arguments |
| `roomId` | string | No | Room context for the tool execution |

```json
{
  "mcpAppId": "app_abc123",
  "toolName": "privos.db.query",
  "arguments": {
    "collection": "contacts"
  },
  "roomId": "room_xyz789"
}
```

### Response

The response depends on the tool being called. See the [PrivOS MCP Tools documentation](./README.md) for tool-specific responses.

```json
{
  "success": true,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "[{\"_id\": \"list_001\", \"name\": \"My List\", ...}]"
      }
    ]
  }
}
```

### Security

- **Scope Enforcement**: The tool's required scope is checked against the app's granted scopes. If the app doesn't have the necessary scope, the call is rejected.
- **Room Context**: When `roomId` is provided, user membership in the room is validated.

### Errors

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `mcpAppId is required` | Missing app ID |
| 400 | `toolName is required` | Missing tool name |
| 403 | `Scope not authorized` | App doesn't have the required scope for this tool |
| 404 | `App not found` | No app with this ID |
| 404 | `Tool not found` | Tool name doesn't exist |

### Example: Create a DB record

```json
{
  "mcpAppId": "app_abc123",
  "toolName": "privos.db.create",
  "arguments": {
    "collection": "contacts",
    "data": { "name": "John Doe", "email": "john@example.com" }
  },
  "roomId": "room_xyz789"
}
```

### Example: Send a bot message

```json
{
  "mcpAppId": "app_abc123",
  "toolName": "privos.bot.sendMessage",
  "arguments": {
    "botToken": "bot_token_here",
    "roomId": "room_xyz789",
    "text": "Hello from MCP app!"
  },
  "roomId": "room_xyz789"
}
```
