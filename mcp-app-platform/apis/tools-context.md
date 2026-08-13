# PrivOS MCP Tools — Context

## `mcpapp.context.get`

Get the current user and room context. No scope required for the base fields;
the `basic:information` identifiers (`appId`, `roomSlug`, `appUrl`) are always
included too.

This tool is also the **reliable delivery path** for the caller's signed
identity: it returns `username` and a short-lived `userToken` (see
[Signed user identity](#signed-user-identity) below). The host additionally
pushes these via `HOST_CONTEXT_CHANGED`, but that push can be missed if it
fires before the iframe attaches its message listener — so always treat this
tool's response as the source of truth.

| | |
|---|---|
| **Scope** | None (public); includes `basic:information` identifiers |

### Arguments

No arguments required.

### Response

Regular room (with default bot still in room):

```json
{
  "userId": "user_123",
  "username": "alice",
  "userToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6...",
  "appId": "6a44fbbb053604eb11d3e30a",
  "roomId": "room_xyz789",
  "roomName": "general",
  "roomSlug": "general",
  "roomType": "c",
  "appUrl": "https://privos-chat-dev.roxane.one/channel/general/mcpapp/6a44fbbb053604eb11d3e30a",
  "isAgentRoom": false,
  "defaultBot": {
    "_id": "bot_abc",
    "username": "general-bot"
  },
  "userRoles": ["owner", "moderator"]
}
```

Agent room:

```json
{
  "userId": "user_123",
  "roomId": "room_xyz789",
  "roomName": "My Agent",
  "roomType": "p",
  "isAgentRoom": true,
  "agentBot": {
    "_id": "bot_abc",
    "username": "my-agent-bot"
  },
  "userRoles": ["owner"]
}
```

Standalone (no room context):

```json
{
  "userId": "user_123",
  "roomId": null
}
```

| Field | Type | Description |
|-------|------|-------------|
| `userId` | string | Current authenticated user ID |
| `username` | string | Current user's username |
| `userToken` | string | Short-lived RS256 JWT proving the caller's identity — verify against the hub JWKS (see [Signed user identity](#signed-user-identity)) |
| `appId` | string | This app's ID — also the `/mcpapp/<appId>` segment of its in-room URL (`basic:information`) |
| `roomId` | string \| null | Current room ID (null when called outside a room) |
| `roomName` | string | Room name (only present when `roomId` is set) |
| `roomSlug` | string | Room slug used in URLs (the room name; only when `roomId` is set) (`basic:information`) |
| `appUrl` | string | Deep link to this app inside the room: `${ROOT_URL}/{channel\|direct\|group}/{roomSlug}/mcpapp/{appId}` (only when `roomId` is set) (`basic:information`) |
| `roomType` | string | Room type: `c` (channel), `p` (private), `d` (DM), `l`/`v` (livechat/voice) |
| `isAgentRoom` | boolean | `true` if the room was created for an agent (set via `customFields.isAgentRoom`) |
| `agentBot` | object \| null | Present only when `isAgentRoom` is `true`. `null` if the agent bot no longer has a subscription in the room |
| `defaultBot` | object \| null | Present only when `isAgentRoom` is `false`. The room's auto-provisioned default bot. `null` if the bot was removed from the room |
| `userRoles` | string[] | User's roles in this room (e.g., `owner`, `moderator`) |

Bot objects contain `_id` and `username`. Both `agentBot` and `defaultBot` are verified against the room's subscriptions — they return `null` if the bot user exists but has been kicked/left the room.

### Example (SDK)

```typescript
const context = await app.callServerTool({
  name: 'mcpapp.context.get',
  arguments: {}
});
// { userId: "user_123", roomId: "room_xyz", roomName: "general", ... }
```

### Example (React Hook)

```tsx
import { useServerTool } from '@anthropic/mcp-react-sdk';

function MyComponent() {
  const { data: context } = useServerTool('mcpapp.context.get');
  return <div>Room: {context?.roomName}</div>;
}
```

## Signed user identity

`userToken` is a short-lived (5 min) RS256 JWT minted by the hub. It lets an app
**backend** verify *who* made a request without being able to forge that identity
— the hub holds the private key; apps only ever fetch the public key.

Claims: `sub` (userId), `preferred_username`, `aud` (appId), `rid` (roomId, when
in a room), standard `iss`/`iat`/`exp`.

Verify it on your backend against the hub JWKS at
`GET /.well-known/mcp-apps/jwks.json` (public keys only). Never trust a
client-supplied `userId` without a token that verifies.

**Frontend → backend flow (relay & direct apps):** read the token from the
context, forward it to your backend, verify, then trust the identity.

```typescript
// frontend (React SDK) — there is no dedicated usePrivosUserToken() hook;
// usePrivosContext() merges the mcpapp.context.get response, so userToken is
// on the returned object even though it isn't in the PrivosContext TS type.
import { usePrivosContext } from '@privos_ai/app-react';
const { userToken } = usePrivosContext() as Record<string, any>;
// send `userToken` to your backend tool call / endpoint

// backend (Node, no JWT lib needed — plain crypto)
import crypto from 'node:crypto';
const jwks = await (await fetch(`${PRIVOS_URL}/.well-known/mcp-apps/jwks.json`)).json();
const [h, p, s] = token.split('.');
const jwk = jwks.keys.find((k) => k.kid === JSON.parse(Buffer.from(h, 'base64url')).kid);
const pub = crypto.createPublicKey({ key: jwk, format: 'jwk' });
const ok = crypto.createVerify('RSA-SHA256').update(`${h}.${p}`).end().verify(pub, Buffer.from(s, 'base64url'));
const claims = JSON.parse(Buffer.from(p, 'base64url').toString());
// ok === true → trust claims.sub / claims.preferred_username
```

> Note: the direct-HTTP tool-call path also forwards the token to the app server
> as `Authorization: Bearer <jwt>` + `X-MCP-User-Id`. The relay (WebSocket)
> transport does not carry per-request headers, so relay app backends obtain the
> token via the frontend (context) as shown above.
