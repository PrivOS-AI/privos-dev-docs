# REST API — Tool Execution

## Execute Tool Call

Execute an MCP tool on a specific app. This is the primary way to invoke Privos MCP tools from the client.

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
  "toolName": "privos.lists.getAll",
  "arguments": {
    "roomId": "room_xyz789"
  },
  "roomId": "room_xyz789"
}
```

### Response

The response depends on the tool being called. See the [Privos MCP Tools documentation](./README.md) for tool-specific responses.

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

### Example: Create a List Item

```json
{
  "mcpAppId": "app_abc123",
  "toolName": "privos.lists.createItem",
  "arguments": {
    "listId": "list_001",
    "title": "John Doe",
    "customFields": [
      { "fieldId": "field_email", "value": "john@example.com" }
    ]
  },
  "roomId": "room_xyz789"
}
```

### Example: Send a Message

```json
{
  "mcpAppId": "app_abc123",
  "toolName": "privos.messages.send",
  "arguments": {
    "roomId": "room_xyz789",
    "text": "Hello from MCP app!"
  },
  "roomId": "room_xyz789"
}
```
