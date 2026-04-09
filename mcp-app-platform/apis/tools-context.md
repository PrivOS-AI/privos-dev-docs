# Privos MCP Tools — Context

## `privos.context.get`

Get the current user and room context. No scope required.

| | |
|---|---|
| **Scope** | None (public) |

### Arguments

No arguments required.

### Response

```json
{
  "userId": "user_123",
  "roomId": "room_xyz789",
  "roomName": "general",
  "roomType": "c",
  "userRoles": ["owner", "moderator"]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `userId` | string | Current authenticated user ID |
| `roomId` | string | Current room ID (from installation context) |
| `roomName` | string | Room name |
| `roomType` | string | Room type: `c` (channel), `p` (private), `d` (DM) |
| `userRoles` | string[] | User's roles in this room (e.g., `owner`, `moderator`) |

### Example (SDK)

```typescript
const context = await app.callServerTool({
  name: 'privos.context.get',
  arguments: {}
});
// { userId: "user_123", roomId: "room_xyz", roomName: "general", ... }
```

### Example (React Hook)

```tsx
import { useServerTool } from '@anthropic/mcp-react-sdk';

function MyComponent() {
  const { data: context } = useServerTool('privos.context.get');
  return <div>Room: {context?.roomName}</div>;
}
```
