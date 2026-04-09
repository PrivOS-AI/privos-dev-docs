# Privos MCP Tools — Rooms

## `privos.rooms.get`

Get room metadata.

| | |
|---|---|
| **Scope** | `rooms:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `roomId` | string | Yes | Room ID |

### Response

```json
{
  "_id": "room_xyz789",
  "name": "general",
  "fname": "General",
  "t": "c",
  "usersCount": 42
}
```

| Field | Type | Description |
|-------|------|-------------|
| `_id` | string | Room ID |
| `name` | string | Room slug name |
| `fname` | string | Room display name |
| `t` | string | Room type: `c` (channel), `p` (private group), `d` (DM) |
| `usersCount` | number | Number of members |

### Example

```typescript
const room = await app.callServerTool({
  name: 'privos.rooms.get',
  arguments: { roomId: 'room_xyz789' }
});
```

---

## `privos.rooms.getMembers`

Get the member list of a room.

| | |
|---|---|
| **Scope** | `rooms:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `limit` | number | No | Max members (default: 50, max: 200) |

### Response

```json
[
  {
    "userId": "user_123",
    "username": "john.doe",
    "roles": ["owner"]
  },
  {
    "userId": "user_456",
    "username": "jane.smith",
    "roles": ["moderator"]
  },
  {
    "userId": "user_789",
    "username": "bob.jones",
    "roles": []
  }
]
```

| Field | Type | Description |
|-------|------|-------------|
| `userId` | string | User ID |
| `username` | string | Username |
| `roles` | string[] | Roles in this room (e.g., `owner`, `moderator`) |

### Example

```typescript
const members = await app.callServerTool({
  name: 'privos.rooms.getMembers',
  arguments: { roomId: 'room_xyz789', limit: 100 }
});
```
