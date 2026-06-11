# PrivOS MCP Tools — Users

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

---

## `privos.users.getByIds`

Despite the name, this tool also accepts `usernames` as a fallback when `userIds` is not provided.


Batch lookup of public profiles by `userIds` **or** `usernames` (one of them, not both).

| | |
|---|---|
| **Scope** | `users:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `userIds` | string[] | One of | List of user IDs (max 100). Takes priority if both fields are provided. |
| `usernames` | string[] | One of | List of usernames (max 100). Used only when `userIds` is empty/missing. |

If both `userIds` and `usernames` are passed, **`userIds` wins** and `usernames` is ignored — there is no merge.

If neither is provided (or both empty), the response is `{ users: [], count: 0 }` with no error.

### Response

```json
{
  "users": [
    { "_id": "user_123", "username": "john.doe", "name": "John Doe", "status": "online", "avatarETag": "abc123" },
    { "_id": "user_456", "username": "jane.doe", "name": "Jane Doe", "status": "offline", "avatarETag": "def456" }
  ],
  "count": 2
}
```

| Field | Type | Description |
|-------|------|-------------|
| `users` | object[] | Public profiles for the resolved users (same shape as `privos.users.get`) |
| `count` | number | Number of users actually returned |

### Behavior

- **Missing entries are silently dropped** — no error if some IDs/usernames don't exist. Compare `count` against your input length to detect missing.
- **Order is not guaranteed** — sort client-side if needed.
- **Limit**: 100 entries per call. Requests beyond the cap throw `Cannot fetch more than 100 users per call`.

### Examples

```typescript
// By IDs
const a = await app.callServerTool({
  name: 'privos.users.getByIds',
  arguments: { userIds: ['user_123', 'user_456'] },
});

// By usernames
const b = await app.callServerTool({
  name: 'privos.users.getByIds',
  arguments: { usernames: ['john.doe', 'jane.doe'] },
});

// Both passed → userIds wins, usernames ignored
const c = await app.callServerTool({
  name: 'privos.users.getByIds',
  arguments: {
    userIds: ['user_123'],
    usernames: ['this-is-ignored'],
  },
});
```
