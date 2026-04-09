# Privos MCP Tools — Users

Only public profile fields are returned for privacy protection.

## `privos.users.get`

Get a user's public profile by ID.

| | |
|---|---|
| **Scope** | `users:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `userId` | string | Yes | User ID |

### Response

```json
{
  "_id": "user_123",
  "username": "john.doe",
  "name": "John Doe",
  "status": "online",
  "avatarETag": "abc123"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `_id` | string | User ID |
| `username` | string | Username |
| `name` | string | Display name |
| `status` | string | Online status: `online`, `away`, `busy`, `offline` |
| `avatarETag` | string | Avatar cache tag (use to construct avatar URL) |

### Example

```typescript
const user = await app.callServerTool({
  name: 'privos.users.get',
  arguments: { userId: 'user_123' }
});
```

---

## `privos.users.getCurrent`

Get the current authenticated user's profile.

| | |
|---|---|
| **Scope** | `users:read` |

### Arguments

No arguments required.

### Response

```json
{
  "_id": "user_123",
  "username": "john.doe",
  "name": "John Doe",
  "status": "online",
  "avatarETag": "abc123"
}
```

### Example

```typescript
const me = await app.callServerTool({
  name: 'privos.users.getCurrent',
  arguments: {}
});
```
