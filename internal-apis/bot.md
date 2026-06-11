# Bot API (Internal)

Internal endpoints for validating bot tokens and verifying bot room access. Designed for server-to-server use (sandbox, agent runtime, relay) where the caller holds a bot token (`privos_...`) and needs to confirm it is still valid before performing an action.

## Base URL

```
/api/v1/internal
```

## Endpoints

### Validate Bot Token

```http
POST /api/v1/internal/bot.validate
```

Validate a bot token, and optionally check whether the bot has access to a specific room.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `token` | string | Yes | The bot token (must start with `privos_`) |
| `roomId` | string | No | If provided, also checks that the bot has a subscription in this room |

**Request:**

```json
{
  "token": "privos_xxxxxxxxxxxxxxxxxxxxxxxx",
  "roomId": "GENERAL"
}
```

**Response (valid):**

```json
{
  "success": true,
  "valid": true
}
```

**Response (invalid):**

```json
{
  "success": true,
  "valid": false,
  "reason": "token_invalid"
}
```

Note: The HTTP status is always **200** when the internal API key is accepted — caller must inspect `valid` (and `reason` when `false`). HTTP **401** is reserved for internal API key failures.

**Reasons:**

| Reason | Meaning |
|--------|---------|
| `token_invalid` | Token does not exist, is inactive, expired, or does not belong to a bot user |
| `bot_inactive` | Token is valid but the bot user is deactivated |
| `room_not_found` | `roomId` was supplied but the room does not exist |
| `no_room_access` | Token is valid but the bot has no subscription in the given room |

**Side effects:**

- Every successful token lookup bumps `lastUsedAt` on the bot token record (same behavior as message-sending endpoints). Treat this endpoint as a real auth call, not a free probe.

**Examples:**

```bash
# Validate token only
curl -X POST "https://your-domain.com/api/v1/internal/bot.validate" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"token":"privos_xxxxxxxxxxxxxxxxxxxxxxxx"}'

# Validate token + room access
curl -X POST "https://your-domain.com/api/v1/internal/bot.validate" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"token":"privos_xxxxxxxxxxxxxxxxxxxxxxxx","roomId":"GENERAL"}'
```

```ts
// fetch
const res = await fetch(`${BASE_URL}/api/v1/internal/bot.validate`, {
  method: 'POST',
  headers: {
    'x-api-key': API_KEY,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ token, roomId }),
});
const { valid, reason } = await res.json();
if (!valid) {
  throw new Error(`Bot validation failed: ${reason}`);
}
```
