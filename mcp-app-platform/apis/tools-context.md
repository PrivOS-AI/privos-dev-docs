# PrivOS MCP Tools — Context

## `mcpapp.context.get`

Get the current user and room context. No scope required for the base fields;
the `basic:information` identifiers (`appId`, `roomSlug`, `appUrl`) are always
included too.

The response also carries `username` and a short-lived `userToken` (see
[Signed user identity](#signed-user-identity) below). `usePrivosContext()` in
`@privos_ai/app-react` does not pass the token on to your UI code, and a UI must
never forward it to a backend as proof of identity.

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
| `userToken` | string | Short-lived RS256 JWT signed by the hub. It is meant for the app backend, which receives it from the hub's dispatch — see [Signed user identity](#signed-user-identity) |
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
import { usePrivosContext } from '@privos_ai/app-react';

function MyComponent() {
  const { roomName } = usePrivosContext();
  return <div>Room: {roomName}</div>;
}
```

## Signed user identity

`userToken` is a short-lived (5 min) RS256 JWT minted by the hub. It lets an app
**backend** verify *who* made a request without being able to forge that identity
— the hub holds the private key; apps only ever fetch the public key.

Claims: `sub` (userId), `preferred_username`, `aud` (appId), `rid` (roomId, when
in a room), standard `iss`/`iat`/`exp`. When the app holds the `rooms:roles:read`
scope the hub also adds the caller's own `room_roles` and `workspace_roles`.

**The UI never forwards a token.** `usePrivosContext()` does not expose it (see
[React SDK › Signed user token](../react-sdk-reference.md#signed-user-token)). The
UI calls your backend through the host bridge, and the hub attaches the verified
caller to the dispatch it sends to your app server:

| App type | Where the backend gets the caller |
|---|---|
| Relay | `params._meta.privosUser.userToken` on each dispatched request, verified against the hub JWKS |
| Direct HTTP | `Authorization: Bearer <jwt>` plus `X-MCP-User-Id` on each dispatched request |
| Managed runtime | the `actor` claim of the signed dispatch assertion, present only when the manifest declares [`capabilities.verifiedActor: true`](../developer-guide.md#declaring-the-verified-actor-capability) |

`@privos_ai/app-server` does the verification for you and surfaces the result as
`context.actor` in your tool handlers. For a paired (standalone) Relay app,
`connectRelay` verifies the relay token automatically; a missing or invalid token
leaves `context.actor` undefined, so check it (`assertActorAvailable`) and refuse
the call rather than falling back to a `userId` argument.

To verify a token yourself, for example in an app-owned HTTP route, use the SDK
instead of hand-rolling the signature check. It pins RS256 and checks the expiry
and the audience:

```typescript
import { buildHubUserTokenAuthOptions, verifyUserToken } from '@privos_ai/app-server';

// The hub publishes its public keys at `/.well-known/mcp-apps/jwks.json`.
const auth = buildHubUserTokenAuthOptions({
  hubOrigin: 'https://<your-hub>',
  audience: '<your app id>', // the token's `aud`
});

const result = await verifyUserToken(token, auth, assertedUserId); // assertedUserId is optional
if (!result.ok) throw new Error(result.message); // fail closed
const { userId, username, roomId } = result.actor;
```
