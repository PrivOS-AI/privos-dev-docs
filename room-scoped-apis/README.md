# Room-Scoped Internal APIs

## Overview

Room-Scoped Internal APIs provide a secure, room-specific way to interact with PrivOS Chat resources. Unlike the standard Internal APIs that use a global API key, Room-Scoped APIs require both an internal API key AND a room-specific session token, ensuring that operations are strictly limited to a specific room context.

**Key Benefits:**
- **Room Isolation**: Each token is bound to a specific room, preventing cross-room access
- **Enhanced Security**: Double authentication with API key + JWT session token
- **Token Management**: Tokens can be created, rotated, and revoked per room
- **Audit Trail**: All operations are tracked with usage count and timestamps
- **Flexible Expiration**: Support for time-limited sessions with auto-regeneration

## Base URL

```
https://your-domain.com/api/v1/internal/rooms/:roomId/*
```

## Authentication Flow

Room-Scoped APIs use a **two-layer authentication** mechanism:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Room-Scoped Authentication Flow                       │
└──────────────────────────────────────────────────────────────────────────────┘

    1. Setup Room Session Key                    2. Generate JWT Token
    ┌──────────────────────┐                    ┌──────────────────────┐
    │ Room Admin creates   │                    │ Server generates JWT │
    │ RoomSessionKey       │──────────────────▶ │ with roomSessionKey  │
    │ (via API or UI)      │                    │ + nonce, signs it    │
    └──────────────────────┘                    └──────────────────────┘
                                                           │
                                                           ▼
    3. API Request with Both Tokens               4. Verification
    ┌──────────────────────┐                    ┌──────────────────────┐
    │ x-api-key:           │──────────────────▶ │ 1. Validate API key  │
    │   INTERNAL_API_KEY   │                    │ 2. Verify JWT        │
    │                      │                    │ 3. Check room match  │
    │ Authorization:       │                    │ 4. Return room data  │
    │   Bearer <JWT_TOKEN> │                    └──────────────────────┘
    └──────────────────────┘
```

### Step 1: Create Room Session Key

First, create a `RoomSessionKey` for the room. This key is stored in MongoDB and can be created via:

**MongoDB Model: `RoomSessionKeys`**

```typescript
{
  _id: string,
  rid: string,              // Room ID
  key: string,              // Random secret key
  active: boolean,          // Active status
  duration: number,         // Duration in seconds (optional)
  expiresAt: Date,          // Expiration timestamp
  autoRegenerate: boolean,  // Auto-regenerate on expiration
  usageCount: number,       // Usage tracking
  rotationCount: number,    // Key rotation count
  createdAt: Date,
  rotatedAt: Date,
  lastUsedAt: Date
}
```

**Creation Example (via MongoDB):**

```javascript
db.room_session_keys.insertOne({
  rid: "ROOM_ID_HERE",
  key: crypto.randomBytes(32).toString('base64'),
  active: true,
  duration: 3600,        // 1 hour
  expiresAt: new Date(Date.now() + 3600000),
  autoRegenerate: true,
  createdAt: new Date(),
  usageCount: 0,
  rotationCount: 0
})
```

### Step 2: Get JWT Token

Once the room session key exists, the server automatically generates a JWT token:

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/{roomId}/session-token" \
  -H "x-api-key: YOUR_INTERNAL_API_KEY"
```

**Response:**

```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresAt": "2024-01-01T12:00:00Z"
  }
}
```

### Step 3: Make API Requests

Use both authentication headers for room-scoped endpoints:

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/{roomId}/lists" \
  -H "x-api-key: YOUR_INTERNAL_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

## Authentication Headers

| Header | Value | Required | Description |
|--------|-------|----------|-------------|
| `x-api-key` | Your internal API key | Yes | Same as standard Internal APIs |
| `Authorization` | `Bearer <JWT_TOKEN>` | Yes | Room-specific JWT token |

## Error Responses

### Missing API Key

```json
{
  "success": false,
  "error": "error-unauthorized",
  "message": "Internal API key required"
}
```

### Invalid API Key

```json
{
  "success": false,
  "error": "error-unauthorized",
  "message": "Invalid Internal API key"
}
```

### Missing/Invalid JWT

```json
{
  "success": false,
  "error": "error-unauthorized",
  "message": "Room session authorization required"
}
```

### Expired JWT

```json
{
  "success": false,
  "error": "error-unauthorized",
  "message": "Invalid or expired room session"
}
```

### Room Not Found

```json
{
  "success": false,
  "error": "error-room-not-found",
  "message": "Room not found"
}
```

## JWT Token Structure

**Payload:**

```typescript
{
  roomSessionKey: string,  // The secret key from RoomSessionKeys collection
  nonce: string,           // Random nonce for uniqueness
  iat: number,            // Issued at timestamp
  exp: number             // Expiration timestamp
}
```

**Configuration:**

- **Algorithm**: HS256
- **Secret**: `ROOM_SESSION_JWT_SECRET` environment variable
- **Expiration**: 1 hour (matches Redis TTL)
- **Storage**: Redis with key `room_session_jwt:{roomId}`

## Token Refresh Logic

The JWT token has a **refresh threshold** of 40 minutes:

| Token Age | Action |
|-----------|--------|
| < 40 minutes | Return existing token, refresh TTL |
| >= 40 minutes | Generate new token with new nonce |
| No RoomSessionKey | Return null |

This ensures tokens are regularly refreshed while maintaining session continuity.

## Room Isolation Guarantee

The authentication middleware enforces room isolation through multiple checks:

```typescript
// 1. Validate API key
if (!validateInternalApiKey(apiKey)) {
  return 401;
}

// 2. Verify JWT and extract roomSessionKey
const roomSessionKey = verifyRoomSessionJWT(jwt);
if (!roomSessionKey) {
  return 401;
}

// 3. Verify room exists
const room = await Rooms.findOneById(roomId);
if (!room) {
  return 404;
}

// 4. Verify JWT matches this room's current session
const currentJWT = await getRoomSessionJWT(roomId);
if (currentJWT && currentJWT !== jwt) {
  return 401; // Token from another room
}

// 5. Attach room context to request
this.roomId = roomId;
this.roomSessionKey = roomSessionKey;
this.room = room;
```

## Available Resources

| Resource | Documentation | Description |
|----------|---------------|-------------|
| **Lists** | [lists.md](./lists.md) | Manage Kanban-style lists in a room |
| **Items** | [items.md](./items.md) | CRUD operations on list items |
| **Stages** | [stages.md](./stages.md) | Manage workflow stages |
| **Documents** | [documents.md](./documents.md) | Document management with versioning |
| **Rooms** | [rooms.md](./rooms.md) | Room info and member listing |
| **Files** | [files.md](./files.md) | MinIO file storage with presigned URLs |
| **AI Chat Sessions** | [ai-chat-sessions.md](./ai-chat-sessions.md) | Context-aware session resume and canvas artifacts management |

## Session Management Endpoints

### Get Session Token

```http
GET /api/v1/internal/rooms/:roomId/session-token
```

**Description:** Get or create the JWT token for a room.

**Headers:**
```
x-api-key: YOUR_INTERNAL_API_KEY
```

**Response:**
```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresAt": "2024-01-01T13:00:00Z"
  }
}
```

### Revoke Session

```http
DELETE /api/v1/internal/rooms/:roomId/session-token
```

**Description:** Revoke the current session token and deactivate the room session key.

**Headers:**
```
x-api-key: YOUR_INTERNAL_API_KEY
Authorization: Bearer YOUR_JWT_TOKEN
```

**Response:**
```json
{
  "success": true,
  "data": {
    "revoked": true,
    "message": "Session revoked successfully"
  }
}
```

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `PRIVOS_CHAT_INTERNAL_API_KEY` | Yes | Internal API key for first-layer auth |
| `ROOM_SESSION_JWT_SECRET` | Yes | Secret key for signing JWT tokens |
| `REDIS_URL` | No | Redis URL (default: `redis://localhost:6379/0`) |

## Best Practices

1. **Secure Storage**: Store JWT tokens securely on the client side
2. **Token Refresh**: Implement automatic refresh before 40-minute threshold
3. **Error Handling**: Handle 401 errors by re-fetching the token
4. **Room Validation**: Always verify room membership before creating session keys
5. **Duration Management**: Set appropriate `duration` based on use case:
   - Short-lived (5-15 min): High-security operations
   - Medium-lived (1 hour): Standard operations
   - Long-lived (24 hours): Background services

## Comparison: Standard vs Room-Scoped APIs

| Feature | Standard Internal APIs | Room-Scoped APIs |
|---------|----------------------|------------------|
| **Authentication** | API Key only | API Key + JWT |
| **Scope** | Global (all resources) | Single room only |
| **Use Case** | Administrative tasks | Room-specific operations |
| **Token Management** | Static API key | Dynamic JWT per room |
| **Revocation** | Change env variable | Per-room revocation |
| **Audit Trail** | Basic | Enhanced with usage tracking |

## Security Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Security Layers                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Layer 1: Network Security                                                  │
│  └─────────────────                                                         │
│  • HTTPS/TLS encryption in transit                                          │
│  • Redis connection security                                                │
│                                                                             │
│  Layer 2: API Key Validation                                                │
│  └───────────────────────────                                               │
│  • Validates internal API key                                               │
│  • Prevents unauthorized server access                                      │
│                                                                             │
│  Layer 3: JWT Token Verification                                            │
│  └─────────────────────────────                                            │
│  • Cryptographic signature verification                                     │
│  • Expiration enforcement                                                  │
│  • Nonce-based uniqueness                                                   │
│                                                                             │
│  Layer 4: Room Session Key Validation                                       │
│  └────────────────────────────────────                                      │
│  • Verifies active RoomSessionKey exists                                    │
│  • Checks key hasn't been revoked                                           │
│  • Validates JWT matches current session in Redis                           │
│                                                                             │
│  Layer 5: Room Context Enforcement                                          │
│  └────────────────────────────────────────                                 │
│  • All operations scoped to single room                                     │
│  • URL-based room isolation (:roomId parameter)                             │
│  • Resource ownership verification (e.g., list.roomId === roomId)           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Troubleshooting

### "Invalid or expired room session"

**Cause:** JWT token is expired or invalid.

**Solution:** Re-fetch the session token from `/session-token` endpoint.

### "Session token does not match this room"

**Cause:** JWT token was generated for a different room.

**Solution:** Ensure you're using the correct token for the room.

### "Room not found"

**Cause:** Room doesn't exist or was deleted.

**Solution:** Verify the room ID and that the room still exists.

### "Internal API key required"

**Cause:** Missing `x-api-key` header.

**Solution:** Include the internal API key in the request headers.

## Support

For issues or questions:
- GitHub Issues: https://github.com/PrivOS-AI/privos-chat/issues
- Documentation: https://docs.privos.ai
