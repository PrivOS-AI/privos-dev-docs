# PrivOS MCP Tools — Messages

## `privos.messages.getRecent`

Get recent messages in a room. User must be a member of the room.

| | |
|---|---|
| **Scope** | `messages:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `limit` | number | No | Max messages (default: 50, max: 100) |

### Response

```json
[
  {
    "_id": "msg_001",
    "msg": "Hello everyone!",
    "u": {
      "_id": "user_123",
      "username": "john.doe",
      "name": "John Doe"
    },
    "ts": "2026-03-25T14:30:00Z"
  },
  {
    "_id": "msg_002",
    "msg": "Welcome to the channel",
    "u": {
      "_id": "user_456",
      "username": "jane.smith",
      "name": "Jane Smith"
    },
    "ts": "2026-03-25T14:31:00Z"
  }
]
```

### Validation

- User must be subscribed to the room (membership is verified)
- `limit` is capped at 100

### Example

```typescript
const messages = await app.callServerTool({
  name: 'privos.messages.getRecent',
  arguments: { roomId: 'room_xyz789', limit: 20 }
});
```

---

## `privos.messages.send`

Send a message to a room. User must be a member of the room.

| | |
|---|---|
| **Scope** | `messages:send` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `text` | string | Yes | Message text content |

### Response

```json
{
  "_id": "msg_003",
  "sent": true
}
```

### Validation

- User must be subscribed to the room (membership is verified)

### Example

```typescript
await app.callServerTool({
  name: 'privos.messages.send',
  arguments: {
    roomId: 'room_xyz789',
    text: 'Task completed successfully!'
  }
});
```
