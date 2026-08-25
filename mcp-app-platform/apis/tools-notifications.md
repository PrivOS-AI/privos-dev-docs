# PrivOS MCP Tools — Notifications

## `mcpapp.notifications.create`

Creates one persistent bell notification and mirrors it to native mobile and Web Push for an active member of the MCP app's approved room.

| Property | Value |
| --- | --- |
| Scope | `notifications:write` |
| Context | `room` |
| Execution | `user`, `background`, or `both` |

Arguments are `userId` (required), `title` (required, 120 characters), `message` (required, 1,000 characters), and optional `actionUrl` (a room-local path, 500 characters). The tool does not accept `roomId`: the Hub uses the server-resolved authorization binding and verifies that the target is active and subscribed to that room.

```ts
await app.callServerTool({
  name: 'mcpapp.notifications.create',
  arguments: { userId: 'target-user-id', title: 'Task assigned', message: 'Review the new task.' },
});
```

The response contains `notificationId`, `userId`, `roomId`, and `type`. Missing consent returns `NOTIFICATIONS_WRITE_SCOPE_REQUIRED`. Push transports are best-effort; a transport failure does not remove the bell record.
