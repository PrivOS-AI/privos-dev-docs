# Agent Chat API

Agent Chat API enables sending messages as bot users and creating in-app notifications.

## Base URL

```
/api/v1/internal/agent-chat
```

## Endpoints

### Send Message as Bot

```http
POST /api/v1/internal/agent-chat.sendMessage
```

Send a message from a bot user to a target user, optionally creating an in-app notification.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | string | Yes | Target user ID |
| `userBotId` | string | Yes | Bot user ID to send from |
| `message` | string | Yes | Message content |
| `isNotification` | boolean | No | Create in-app notification (default: false) |

**Request:**

```json
{
  "userId": "TARGET_USER_ID",
  "userBotId": "reminder.bot",
  "message": "You have 3 pending tasks that need attention",
  "isNotification": true
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": {
      "_id": "MSG_ID",
      "rid": "DIRECT_ROOM_ID",
      "msg": "You have 3 pending tasks that need attention",
      "u": {
        "_id": "reminder.bot",
        "username": "reminder.bot",
        "name": "Reminder Bot"
      },
      "ts": "2024-01-15T10:00:00.000Z"
    },
    "room": {
      "_id": "DIRECT_ROOM_ID",
      "t": "d",
      "users": ["TARGET_USER_ID", "reminder.bot"]
    }
  }
}
```

---

## Behavior

### Direct Room Creation

If a direct room between the target user and bot doesn't exist, it will be automatically created before sending the message.

### In-App Notifications

When `isNotification: true`, an in-app notification is created with:

**For `reminder.bot`:**
- Creates a special reminder summary notification
- Includes task count extracted from message
- Links to the direct message

**For other bots:**
- Creates a generic bot notification
- Links to the direct message

### Notification Flow

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Bot API   │────▶│ Bot Sends    │────▶│ Target User │
│   Endpoint  │     │   Message    │     │  Receives   │
└─────────────┘     └──────────────┘     │   Message   │
                                           └─────────────┘
                                                    │
                                                    ▼
                                      ┌─────────────────────┐
                                      │ In-App Notification │
                                      │     (Optional)       │
                                      └─────────────────────┘
```

---

## Bot User Requirements

The bot user must:
1. Exist in the system
2. Have `type: 'bot'`
3. Be properly configured

If the bot user doesn't exist or isn't a bot, the API returns an error.

```json
{
  "success": false,
  "error": "Bot user BOT_ID not found"
}
```

```json
{
  "success": false,
  "error": "User BOT_ID is not a bot user"
}
```

---

## Special Bot: reminder.bot

The `reminder.bot` has special handling:
- **Title**: "Reminder bot notification"
- **Summary Type**: "daily" (default)
- **Task Count Extraction**: Automatically extracts number from message pattern "(\d+)\s*(?:task|tasks)"

**Example Messages:**

```json
{
  "message": "You have 5 tasks due today"
}
// Extracts: taskCount = 5
```

---

## Error Responses

| Error Code | Description |
|------------|-------------|
| `error-invalid-params` | Missing required parameters (userId, userBotId, message) |
| `error-user-not-found` | Target user not found |
| `error-bot-not-found` | Bot user not found |
| `error-invalid-bot` | Specified user is not a bot user |
| `error-message-required` | Message content is empty |

---

## ISendMessageParams Interface

```typescript
interface ISendMessageParams {
  userId: string;          // Target user ID
  userBotId: string;        // Bot user ID
  message: string;          // Message content
  isNotification?: boolean; // Create notification
}

interface ISendMessageResponse {
  success: boolean;
  message?: IMessage;
  room?: IRoom;
}

interface IMessage {
  _id: string;
  rid: string;              // Room ID
  msg: string;              // Message content
  u: {                     // Sender
    _id: string;
    username: string;
    name?: string;
  };
  ts: Date;                 // Timestamp
}
```

---

## Use Cases

### 1. Daily Reminders

```http
POST /api/v1/internal/agent-chat.sendMessage
```

```json
{
  "userId": "USER_ID",
  "userBotId": "reminder.bot",
  "message": "You have 7 tasks due today. Please review your backlog.",
  "isNotification": true
}
```

### 2. AI Assistant Updates

```http
POST /api/v1/internal/agent-chat.sendMessage
```

```json
{
  "userId": "USER_ID",
  "userBotId": "ai-assistant",
  "message": "Your document has been generated and is ready for review.",
  "isNotification": true
}
```

### 3. System Notifications

```http
POST /api/v1/internal/agent-chat.sendMessage
```

```json
{
  "userId": "USER_ID",
  "userBotId": "system.bot",
  "message": "Your workspace export is complete.",
  "isNotification": true
}
```

---

## Related Resources

- [Agent Chat Session API](./agent-chat-session.md) - Canvas artifact management
- [Users API](./users.md) - User information
- [Rooms API](./rooms.md) - Direct room creation
