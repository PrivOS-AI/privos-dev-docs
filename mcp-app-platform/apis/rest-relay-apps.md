# REST API — Relay Apps

## Generate Pairing URL

Generate a one-time pairing URL for relay app developers to use during setup.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/mcp-apps.generate-pair-url` |
| **Auth** | Admin (`manage-oauth-apps` permission) |

### Request

No body required. Uses the authenticated user's context.

### Response

```json
{
  "success": true,
  "pairUrl": "https://chat.privos.com/pair?token=eyJ...",
  "pairToken": "pair_abc_123xyz"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `pairUrl` | string | Full URL to share with relay app developer |
| `pairToken` | string | Token to poll pairing status |

### How It Works

1. Admin generates a pairing URL
2. Share the `pairUrl` with the relay app developer
3. Developer runs `npm start` with the pairing URL
4. Poll `mcp-apps.pair-status` to check when pairing completes
5. Once paired, the response includes `clientId`, `clientSecret`, and `relayUrl`

---

## Check Pairing Status

Poll this endpoint to check if a relay app has completed the pairing flow.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/mcp-apps.pair-status` |
| **Auth** | Admin |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | string (query) | Yes | The pairing token from `generate-pair-url` |

```
GET /api/v1/mcp-apps.pair-status?token=pair_abc_123xyz
```

### Response — Waiting

```json
{
  "success": true,
  "status": "waiting",
  "createdAt": "2026-03-25T10:00:00Z",
  "expiresAt": "2026-03-25T11:00:00Z"
}
```

### Response — Paired

```json
{
  "success": true,
  "status": "paired",
  "app": {
    "_id": "app_123",
    "name": "My Relay App",
    "clientId": "client_abc",
    "clientSecret": "secret_xyz",
    "relayUrl": "wss://chat.privos.com/api/v1/mcp-apps.relay"
  }
}
```

### Response — Expired

```json
{
  "success": true,
  "status": "expired"
}
```

| Status Value | Description |
|--------------|-------------|
| `waiting` | Token is valid, waiting for relay app to connect |
| `paired` | Relay app has successfully paired |
| `expired` | Token has expired (default: 1 hour) |

### Errors

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `token is required` | Missing pairing token |

---

## Check Relay Status

Check if a relay app is currently online (WebSocket connected).

| | |
|---|---|
| **Endpoint** | `GET /api/v1/mcp-apps.relay-status` |
| **Auth** | User |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mcpAppId` | string (query) | Yes | App ID to check |

```
GET /api/v1/mcp-apps.relay-status?mcpAppId=app_123
```

### Response

```json
{
  "success": true,
  "online": true
}
```

| Field | Type | Description |
|-------|------|-------------|
| `online` | boolean | Whether the relay app has an active WebSocket connection |

### Errors

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `mcpAppId is required` | Missing app ID |

---

## OAuth Token (for relay apps)

Relay apps use OAuth2 Client Credentials flow to obtain access tokens for the WebSocket connection.

| | |
|---|---|
| **Endpoint** | `POST /oauth/token` |
| **Auth** | Client Credentials |
| **Content-Type** | `application/x-www-form-urlencoded` |

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `grant_type` | string | Yes | Must be `client_credentials` |
| `client_id` | string | Yes | From pairing response |
| `client_secret` | string | Yes | From pairing response |

```bash
curl -X POST https://chat.privos.com/oauth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=client_abc&client_secret=secret_xyz"
```

### Response

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

| Field | Type | Description |
|-------|------|-------------|
| `access_token` | string | JWT token for WebSocket authorization |
| `token_type` | string | Always `Bearer` |
| `expires_in` | number | Token validity in seconds |

### Usage

Use the access token in the WebSocket connection's Authorization header:

```
wss://chat.privos.com/api/v1/mcp-apps.relay
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## WebSocket Relay Endpoint

| | |
|---|---|
| **Endpoint** | `wss://{host}/api/v1/mcp-apps.relay` |
| **Auth** | OAuth Bearer token in `Authorization` header |

The WebSocket connection is used for bidirectional JSON-RPC communication between the PrivOS server and the relay app. Once connected, the relay app receives discovery and resource requests such as `initialize`, `tools/list`, and `resources/read`, and can also receive `tools/call` requests when PrivOS executes server-side tools on behalf of the relay app. The relay app replies with standard JSON-RPC responses.
