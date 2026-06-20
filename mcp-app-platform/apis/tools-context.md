# PrivOS MCP Tools — Context

## `mcpapp.context.get`

Get the current user and room context. No scope required.

| | |
|---|---|
| **Scope** | None (public) |

### Arguments

No arguments required.

### Response

Regular room (with default bot still in room):

```json
{
  "userId": "user_123",
  "roomId": "room_xyz789",
  "roomName": "general",
  "roomType": "c",
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
| `roomId` | string \| null | Current room ID (null when called outside a room) |
| `roomName` | string | Room name (only present when `roomId` is set) |
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
