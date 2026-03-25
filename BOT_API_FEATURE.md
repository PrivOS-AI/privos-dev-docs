# Privos Chat - Bot API Feature

A Telegram Bot API-like feature for Privos Chat that enables developers to create and manage bots with API tokens, webhooks, and custom functions.

## Overview

This implementation provides a complete Bot API system similar to Telegram's Bot API, allowing:
- Bot token authentication
- Message operations (send, edit, delete)
- Webhook subscriptions for real-time events
- Custom bot functions/commands
- Redis-based queue for reliable webhook delivery

## Features

### 1. Bot Token System

- **Token Format**: `privos_<userId>_<secret>`
- **Auto-generated**: A unique 64-character hex secret
- **Permissions**: `read`, `write`, `admin`
- **Scopes**: `messages:read`, `messages:write`, etc.
- **Expiration**: Optional expiration date support
- **Active/Inactive**: Tokens can be deactivated without deletion

### 2. Message Operations

- **Send Message**: Send messages as a bot to any room
- **Edit Message**: Edit previously sent messages
- **Delete Message**: Delete messages sent by the bot
- **Room-based**: Operations target specific rooms by ID

### 3. Webhook System

- **Event Filtering**: Subscribe to specific events (18 event types available)
- **Room-based Filtering**: Only receive events from rooms where the bot is added
- **HMAC Signatures**: Webhook payloads signed with SHA256 for security
- **Retry Logic**: Automatic retry with exponential backoff (up to 24 attempts)
- **Redis Queue**: BullMQ-based queue for reliable delivery

#### Supported Event Types

18 total event types across 6 categories:

- **Message Events** (4): `message.new`, `message.edited`, `message.deleted`, `message.mention`
- **Room Events** (2): `room.joined`, `room.left`
- **User Events** (2): `user.joined`, `user.left`
- **List Item Events** (4): `list.item.created`, `list.item.deleted`, `list.item.stage_changed`, `list.item.attributes_changed`
- **File Events** (3): `file.created`, `file.updated`, `file.deleted`
- **Folder Events** (3): `folder.created`, `folder.deleted`, `folder.renamed`

See [Supported Webhook Events Reference](./bot-webhooks/supported-webhook-events-reference.md) for complete event details and payload structures.

**Important**: Bot messages are excluded from triggering webhooks to prevent infinite loops. Bots only receive events from regular users.

### 4. Bot Functions

- **Custom Commands**: Define custom bot functions/commands
- **HTTP Handlers**: Function handlers can be HTTP endpoints
- **Discovery**: Functions can be discovered and invoked via API

### 5. Bot Management UI

- **Create Bot**: Auto-generates API token on creation
- **Manage Tokens**: View, create, and invalidate tokens
- **Manage Webhooks**: Configure webhooks for events
- **Manage Functions**: Define custom bot functions
- **Bot Info Modal**: Access from user card for bot users

### 6. Inline Keyboard

- **Keyboard Buttons**: Messages can have interactive inline keyboard buttons displayed below the message content
- **Button Types**: Supports multiple action types (callback, url, copy, message, bot-event)
- **Keyboard Types**: Actions keyboards, single-select, and multi-select keyboards
- **Visibility States**: Buttons can be active, inactive, or hidden
- **One-Time**: Keyboards can be configured to collapse after first interaction
- **Expiration**: Optional keyboard expiration timestamps
- **bot-event Action**: Special button action that triggers existing BotWebhookEvent webhooks with custom metadata
- **Style Options**: Buttons support styling, emojis, and icons

## Database Schema

### Bot Tokens Collection (`rocketchat_bot_tokens`)

```typescript
{
  _id: string;
  token: string;              // privos_<userId>_<secret>
  userId: string;            // Bot user ID
  botUsername: string;        // Bot username
  secret: string;            // 64-char hex
  name: string;              // Token name
  description?: string;      // Optional description
  permissions: string[];     // read, write, admin
  scopes: string[];          // messages:read, messages:write, etc.
  active: boolean;           // true/false
  expiresAt?: Date;          // Optional expiration
  lastUsedAt?: Date;         // Last usage timestamp
  createdAt: Date;
  _updatedAt: Date;
}
```

### Bot Webhooks Collection (`rocketchat_bot_webhooks`)

```typescript
{
  _id: string;
  id: string;                // Webhook ID
  botUserId: string;         // Bot user ID
  url: string;               // Webhook URL
  events: string[];          // Event types
  secret: string;            // HMAC secret
  roomIds?: string[];        // Specific rooms (optional)
  active: boolean;
  createdAt: Date;
  _updatedAt: Date;
}
```

### Bot Functions Collection (`rocketchat_bot_functions`)

```typescript
{
  _id: string;
  id: string;                // Function ID
  botUserId: string;         // Bot user ID
  name: string;              // Function name
  description: string;       // Function description
  handler: string;           // HTTP handler URL
  createdAt: Date;
  _updatedAt: Date;
}
```

## API Endpoints

### Authentication

All Bot API endpoints require authentication via the `Authorization` header:

```
Authorization: Bearer privos_<userId>_<secret>
```

### Bot Operations

#### Create Bot
```
POST /api/v1/bots.create
Content-Type: application/json

{
  "username": "mybot",
  "name": "My Bot",
  "type": "bot",
  "roles": ["bot"]
}
```

**Response**: Auto-creates and returns a bot token
```json
{
  "bot": { ... },
  "token": "privos_<userId>_<secret>",
  "message": "Bot created successfully"
}
```

**Automatic Greeting Message**:
When a new bot is created, it automatically sends a greeting message to the bot owner in a direct message. The greeting includes:
- Bot introduction with name
- Auto-generated API token for easy integration
- Instructions on keeping the token secure
- Information about managing tokens, webhooks, and functions

The greeting message format:
```
Hello! I'm **{bot_name}**, your new bot. 🤖

Here's my API token for integration:
`{token}`

Keep this token secure - it gives full access to my bot capabilities.

You can manage my tokens, webhooks, and functions from the bot info panel.
```

### Token Operations

#### Generate Token
```
POST /api/v1/bot.tokens.generate
Authorization: Bearer <token>
Content-Type: application/json

{
  "botUserId": "<bot_user_id>",
  "botUsername": "mybot",
  "name": "Production Token",
  "description": "API token for production"
}
```

#### List Tokens
```
GET /api/v1/bot.tokens.list?botUserId=<bot_user_id>
Authorization: Bearer <token>
```

#### Invalidate Token
```
POST /api/v1/bot.tokens.invalidate
Authorization: Bearer <token>
Content-Type: application/json

{
  "tokenId": "<token_id>"
}
```

### Message Operations

#### Send Message
```
POST /api/v1/bot/sendMessage
Authorization: Bearer <bot_token>
Content-Type: application/json

{
  "roomId": "<room_id>",
  "text": "Hello from bot!",
  "reply_to_message_id": "<message_id>",  // Optional: Reply to a message
  "inlineKeyboard": {                      // Optional: Inline keyboard buttons
    "type": "actions",                     // Type: actions, single_select, multi_select
    "buttons": [
      {
        "id": "button1",
        "text": "Approve",
        "action": "bot-event",             // Action type: callback, url, copy, message, bot-event
        "actionData": "{\"event\": \"message.mention\", \"botUserId\": \"bot123\", \"metadata\": {\"action\": \"approve\"}}"
      }
    ],
    "submitButton": {                      // Optional: Submit button for select keyboards
      "text": "Submit"
    },
    "visibility": "active",                // Optional: active, inactive, hidden
    "oneTime": true,                       // Optional: Collapse after interaction
    "expiresAt": "2024-01-31T23:59:59.999Z" // Optional: Expiration timestamp
  }
}
```

#### Send Photo
Send a photo to a room using `multipart/form-data`. The `photo` field can be either a file or a URL string.

**Request:**
```
POST /api/v1/bot/sendPhoto
Authorization: Bearer <bot_token>
Content-Type: multipart/form-data

roomId: <room_id>
photo: <file_or_url>             # Photo file OR URL string
caption: <text>                  # Optional: Photo caption
text: <text>                     # Optional: Message text (takes priority over caption)
reply_to_message_id: <message_id> # Optional: Reply to a message
```

**Photo field options:**
- **File**: Upload an image file directly
- **URL string**: Provide a publicly accessible image URL

**Example using curl (file upload):**
```bash
curl -X POST http://localhost:3000/api/v1/bot/sendPhoto \
  -H "Authorization: Bearer privos_abc123..." \
  -F "roomId=GENERAL" \
  -F "photo=@/path/to/image.jpg" \
  -F "caption=Check this out!"
```

**Example using curl (URL string):**
```bash
curl -X POST http://localhost:3000/api/v1/bot/sendPhoto \
  -H "Authorization: Bearer privos_abc123..." \
  -F "roomId=GENERAL" \
  -F "photo=https://example.com/image.jpg" \
  -F "caption=Check this out!"
```

#### Send Audio
Send an audio file to a room. **Only file upload is supported** (no URL).

```
POST /api/v1/bot/sendAudio
Authorization: Bearer <bot_token>
Content-Type: multipart/form-data

roomId: <room_id>
audio: <file>                    # Required: Audio file (.mp3 or .m4a only)
reply_to_message_id: <message_id> # Optional: Reply to a message
```

**Supported formats**: `.mp3`, `.m4a` only

#### Send Video
Send a video to a room. **Only file upload is supported** (no URL).

```
POST /api/v1/bot/sendVideo
Authorization: Bearer <bot_token>
Content-Type: multipart/form-data

roomId: <room_id>
video: <file>                    # Required: Video file
reply_to_message_id: <message_id> # Optional: Reply to a message
```

#### Send Voice
Send a voice message to a room. **Only file upload is supported** (no URL).

```
POST /api/v1/bot/sendVoice
Authorization: Bearer <bot_token>
Content-Type: multipart/form-data

roomId: <room_id>
voice: <file>                    # Required: Voice file
reply_to_message_id: <message_id> # Optional: Reply to a message
```

**Supported formats**: `.ogg`, `.oga`, `.opus`, `.mp3`, `.webm`

#### Send Document
Send a document to a room. **Only file upload is supported** (no URL). Max 50MB.

```
POST /api/v1/bot/sendDocument
Authorization: Bearer <bot_token>
Content-Type: multipart/form-data

roomId: <room_id>
document: <file>                 # Required: Document file (max 50MB)
reply_to_message_id: <message_id> # Optional: Reply to a message
```

#### Edit Message
```
POST /api/v1/bot/editMessage
Authorization: Bearer <bot_token>
Content-Type: application/json

{
  "messageId": "<message_id>",
  "text": "Updated message"
}
```

#### Delete Message
```
POST /api/v1/bot/deleteMessage
Authorization: Bearer <bot_token>
Content-Type: application/json

{
  "messageId": "<message_id>"
}
```

### Webhook Operations

#### Set Webhook (Telegram Bot API style)
```
POST /api/v1/bot/setWebhook
Authorization: Bearer <bot_token>
Content-Type: application/json

{
  "url": "https://your-server.com/webhook",
  "events": [
    "message.new", "message.edited", "message.deleted", "message.mention",
    "room.joined", "room.left",
    "user.joined", "user.left",
    "list.item.created", "list.item.deleted", "list.item.stage_changed", "list.item.attributes_changed",
    "file.created", "file.updated", "file.deleted",
    "folder.created", "folder.deleted", "folder.renamed"
  ],
  "secret": "webhook_secret"  // optional, auto-generated if not provided
}
```

**Available Events**: 18 total event types. See [Supported Webhook Events Reference](./bot-webhooks/supported-webhook-events-reference.md) for complete list and descriptions.

**Note**: The `botUserId` is automatically detected from the bot token in the Authorization header. No need to specify it in the request body.

#### Get Webhook Info (Telegram Bot API style)
```
GET /api/v1/bot/getWebhookInfo
Authorization: Bearer <bot_token>
```

**Response** (with webhook configured):
```json
{
  "id": "webhook_id",
  "url": "https://your-server.com/webhook",
  "events": ["message.new", "message.edited"],
  "roomIds": ["room1", "room2"],
  "active": true,
  "lastTriggeredAt": "2024-01-01T00:00:00.000Z",
  "failureCount": 0,
  "createdAt": "2024-01-01T00:00:00.000Z"
}
```

**Response** (no webhook configured):
```json
{
  "url": "",
  "hasCustomCertificate": false,
  "pendingUpdateCount": 0,
  "lastError": null
}
```

#### Delete Webhook (Telegram Bot API style)
```
POST /api/v1/bot/deleteWebhook
Authorization: Bearer <bot_token>
```

**Note**: One bot can only have one webhook. Calling `/setWebhook` will replace any existing webhook.

### Inline Keyboard Callback Operations

#### Keyboard Button Click Callback
```
POST /api/v1/chat.keyboardCallback
Content-Type: application/json

{
  "messageId": "<message_id>",
  "roomId": "<room_id>",
  "buttonId": "<button_id>",
  "actionData": "<optional_action_data>",  // Optional: Action data from button
  "keyboardType": "actions",               // Type: actions, single_select, multi_select
  "selectedValues": ["value1", "value2"]   // Optional: For multi-select keyboards
}
```

**Response**:
```json
{
  "success": true,
  "message": "Callback processed"
}
```

**Notes**:
- For `bot-event` action buttons, the button click triggers a webhook delivery using the existing BotWebhookEvent system
- The `actionData` contains a JSON string with `event`, `botUserId`, and `metadata` fields
- The bot's webhook receives the callback with `metadata` included in the `form` field
- For `single_select` and `multi_select` keyboards, `selectedValues` contains the user's selection(s)

**Example bot-event Button**:

A button configured with:
```json
{
  "id": "approve_btn",
  "text": "Approve Request",
  "action": "bot-event",
  "actionData": "{\"event\": \"message.mention\", \"botUserId\": \"bot_id_123\", \"metadata\": {\"action\": \"approve\", \"requestId\": \"req_001\"}}"
}
```

When clicked, the bot's webhook receives:
```json
{
  "event": "message.mention",
  "timestamp": "2024-01-05T11:30:00.000Z",
  "bot": { ... },
  "form": {
    "roomId": "room_id",
    "roomName": "general",
    "roomSessionKey": "...",
    "privosEndpointUrl": "...",
    "metadata": {
      "action": "approve",
      "requestId": "req_001"
    }
  },
  "message": { ... }
}
```

### Function Operations

#### Create Function
```
POST /api/v1/bot.functions.create
Authorization: Bearer <bot_token>
Content-Type: application/json

{
  "botUserId": "<bot_user_id>",
  "name": "get_weather",
  "description": "Get current weather",
  "handler": "https://your-server.com/functions/weather"
}
```

#### List Functions
```
GET /api/v1/bot.functions.list?botUserId=<bot_user_id>
Authorization: Bearer <bot_token>
```

#### Delete Function
```
POST /api/v1/bot.functions.delete
Authorization: Bearer <bot_token>
Content-Type: application/json

{
  "functionId": "<function_id>"
}
```

## Webhook Payload Format

Webhooks receive POST requests with the following format:

### Headers

- `X-Privos-Signature`: HMAC SHA256 signature of payload
- `Content-Type`: application/json

### Basic Structure

```json
{
  "event": "<event_type>",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "bot": { "id": "...", "username": "...", "name": "..." },
  "room": { "id": "...", "name": "...", "type": "..." },
  "message": { ... },  // For message events
  "user": { ... },     // For user events
  "item": { ... },     // For list item events
  "file": { ... },     // For file events
  "folder": { ... },   // For folder events
  "actor": { ... }     // For list/file/folder events
}
```

See [Detailed Payload Examples](./bot-webhooks/webhook-payload-examples.md) for complete JSON examples of all event types.

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `REDIS_URL` | Redis connection URL for webhook queue | Required |
| `WEBHOOK_RETRY_ATTEMPTS` | Maximum retry attempts for failed webhooks | 24 |
| `WEBHOOK_RETRY_INITIAL_DELAY` | Initial retry delay in ms | 30000 |
| `WEBHOOK_RETRY_MAX_DELAY` | Maximum retry delay in ms | 600000 |

## File Structure

```
privos-chat/
├── packages/
│   ├── core-typings/src/
│   │   └── BotTokens.ts          # Type definitions
│   └── models/src/
│       └── models/
│           ├── BotTokens.ts       # MongoDB model
│           ├── BotWebhooks.ts     # MongoDB model
│           └── BotFunctions.ts    # MongoDB model
├── apps/meteor/
│   ├── app/
│   │   ├── api/server/v1/
│   │   │   ├── bot-api.ts        # REST API endpoints
│   │   │   └── bots.ts          # Modified to auto-create token
│   │   └── bot-api/
│   │       └── server/
│   │           └── index.ts      # Bot services (Token, Webhook, Function, Message)
│   ├── app/
│   │   └── bot-webhook-queue/
│   │       └── server/
│   │           └── index.ts      # Webhook queue with Redis + BullMQ
│   ├── client/
│   │   ├── components/
│   │   │   ├── BotCreatedSuccess/  # Bot creation success modal
│   │   │   └── BotInfo/            # Bot info (tokens, webhooks, functions)
│   │   └── views/room/
│   │       ├── modals/
│   │       │   └── BotInfoModal/  # Bot management modal
│   │       └── hooks/
│   │           └── useUserInfoActions/
│   │               └── actions/
│   │                   └── useManageBotAction.tsx  # "Manage Bot" action
│   └── server/
│       ├── configuration/
│       │   └── bot.ts             # Bot configuration
│       └── models.ts              # Model registrations
```

## Usage Example

### 1. Create a Bot

When creating a bot via the UI (`+` → `Bot`), a token is automatically generated:

```json
{
  "token": "privos_abc123def4567890123456789012345678901234567890123456789012345678"
}
```

### 2. Send a Message as Bot

```bash
curl -X POST http://localhost:3000/api/v1/bot/sendMessage \
  -H "Authorization: Bearer privos_abc123..." \
  -H "Content-Type: application/json" \
  -d '{
    "roomId": "GENERAL",
    "text": "Hello from my bot!"
  }'
```

### 3. Set Up a Webhook

```bash
curl -X POST http://localhost:3000/api/v1/bot/setWebhook \
  -H "Authorization: Bearer privos_abc123..." \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://myserver.com/webhooks/bot",
    "events": ["message.new"]
  }'
```

**Note**: The `secret` field is optional. If not provided, a random secret will be auto-generated and returned in the response. The `botUserId` is automatically detected from the bot token.

### 4. Verify Webhook Signature (Node.js)

```javascript
const crypto = require('crypto');
const signature = req.headers['x-privos-signature'];
const hmac = crypto.createHmac('sha256', 'my_webhook_secret');
hmac.update(req.body);
const digest = hmac.digest('hex');

if (signature !== digest) {
  return res.status(401).send('Invalid signature');
}
```

## Redis Queue Configuration

The webhook queue uses Redis + BullMQ for reliable delivery:

```javascript
// Queue configuration
const queue = new Queue('bot-webhooks', {
  connection: {
    host: 'localhost',
    port: 6379,
    password: 'privos-redis-dev',
  },
});
```

### Retry Strategy

The webhook queue uses **BullMQ** with built-in retry support and exponential backoff:

| Setting | Environment Variable | Default Value |
|---------|---------------------|---------------|
| Max Retries | `WEBHOOK_RETRY_ATTEMPTS` | **24 attempts** |
| Initial Delay | `WEBHOOK_RETRY_INITIAL_DELAY` | **30,000ms (30 seconds)** |
| Max Delay | `WEBHOOK_RETRY_MAX_DELAY` | **600,000ms (10 minutes)** |
| Backoff Type | - | **Exponential** |
| Concurrency | - | **1 (strict FIFO ordering)** |

#### Retry Timeline (Exponential Backoff)

```
Attempt 1:  Immediate (0s)
Attempt 2:  30 seconds
Attempt 3:  60 seconds
Attempt 4:  120 seconds (2 min)
Attempt 5:  240 seconds (4 min)
Attempt 6:  480 seconds (8 min)
...
Attempt 24: ~23 days
```

**Total coverage**: With exponential backoff, webhooks are retried for approximately 24 hours before being permanently failed.

#### Behavior on Failure

- **Temporary failures** (5xx, network errors): Automatic retry with exponential backoff
- **Permanent failures** (4xx except 429): Job is marked as failed immediately (no retry)
- **Rate limiting** (429): Retried with exponential backoff
- **Webhook inactive**: Job fails immediately without retry

## Architecture Decisions

### Why Redis + BullMQ?

- **Reliability**: Webhooks persist in queue until delivered
- **Scalability**: Multiple workers can process jobs in parallel
- **Retry Logic**: Built-in exponential backoff
- **Monitoring**: Queue metrics and job status tracking

### Why Room-based Webhook Filtering?

- **Privacy**: Bots only receive events from rooms they're added to
- **Performance**: Fewer unnecessary webhook deliveries
- **Security**: Limits bot's event visibility

### Why Bot Token Format `privos_<userId>_<secret>`?

- **User Association**: Easy to identify which user owns the bot
- **Uniqueness**: User ID + random secret ensures uniqueness
- **Prefix**: Easy to identify as Privos Chat tokens
- **Entropy**: 64-char hex provides 256 bits of randomness

## Comparison with Telegram Bot API

| Feature | Privos Chat Bot API | Telegram Bot API |
|---------|---------------------|-------------------|
| **Token Format** | `privos_<userId>_<secret>` | `<bot_id>:<secret>` |
| **Webhook Retention** | ~24 hours (via Redis queue) | 24 hours (Telegram server) |
| **Retry Strategy** | Exponential backoff, 24 attempts | Continuous until 24h expires |
| **Event Types** | 18 events (message, room, user, list, file, folder) | Updates (messages, callbacks, etc.) |
| **Webhook Signature** | HMAC SHA256 | SHA256 |
| **Room Filtering** | Built-in (bot only sees events from rooms it's in) | No filtering (bot sees all messages) |
| **Message Operations** | Send, Edit, Delete | Send, Edit, Delete |
| **Media Support** | Photo, Audio, Video, Voice, Document | Photo, Audio, Video, Voice, Document |
| **Queue Persistence** | Redis + BullMQ | Telegram server (proprietary) |

### Key Differences

#### Event Filtering
- **Privos Chat**: Bots only receive events from rooms they are added to. More secure and private.
- **Telegram**: Bot receives ALL messages sent to it, must filter client-side.

#### Webhook Reliability
- **Privos Chat**: Uses persistent Redis queue with exponential backoff. Survives server restarts.
- **Telegram**: Webhooks stored on Telegram servers for max 24h. Lost if not delivered in time.

#### Event Types Comparison
| Privos Chat | Telegram | Description |
|-------------|----------|-------------|
| `message.*` (4) | `message`, `edited_message` | Message events |
| `room.*` (2) | `my_chat_member` | Room membership |
| `user.*` (2) | `chat_member` | User membership |
| `list.item.*` (4) | N/A | List management |
| `file.*` (3) | N/A | File management |
| `folder.*` (3) | N/A | Folder management |

Privos Chat supports 10 additional event types beyond Telegram's core functionality.

## Security Considerations

1. **Token Storage**: Tokens stored with hashed secrets
2. **Webhook Signatures**: HMAC SHA256 prevents tampering
3. **Permissions**: Scopes limit token capabilities
4. **Rate Limiting**: Consider rate limiting bot API endpoints
5. **HTTPS**: Always use HTTPS for webhooks in production

## Troubleshooting

### Bot Token Not Found

- Verify token format: `privos_<userId>_<secret>`
- Check token is `active: true`
- Ensure bot user exists and has `type: 'bot'`

### Webhooks Not Delivering

- Check Redis is running: `docker ps | grep redis`
- Verify webhook URL is accessible
- Check logs for delivery errors
- Monitor queue: BullBoard for Redis queues

### "Model IBotTokensModel not found"

- Ensure models are registered in `apps/meteor/server/models.ts`
- Check `BotTokensRaw`, `BotWebhooksRaw`, `BotFunctionsRaw` are imported and registered

## Future Enhancements

- Bot commands, payments, rate limiting, analytics, marketplace discovery
