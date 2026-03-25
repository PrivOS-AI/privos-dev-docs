# Bot Webhook API Documentation

This document describes the webhook payload format and authentication for handling bot webhooks from Privos Chat.

## Webhook Authentication

Webhooks use **secret token authentication** (similar to Telegram's approach):

### Secret Token Header (Required)

Every webhook request includes a `X-Privos-Bot-Api-Secret-Token` header:

```
X-Privos-Bot-Api-Secret-Token: your-random-secret-token
```

This provides a simple static token for basic authentication. The token is configured in your bot settings and should be kept secure.

**Example (Node.js/Express):**

```javascript
app.post('/webhook', (req, res) => {
    const secretToken = req.headers['x-privos-bot-api-secret-token'];
    const expectedToken = process.env.WEBHOOK_SECRET_TOKEN; // Store in environment variable

    if (secretToken !== expectedToken) {
        return res.status(401).send('Invalid token');
    }

    // Process webhook
    handleWebhook(req.body);
    res.status(200).send('OK');
});
```

**Example (Python/Flask):**

```python
import os
from flask import request, Flask

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    secret_token = request.headers.get('X-Privos-Bot-Api-Secret-Token')
    expected_token = os.environ.get('WEBHOOK_SECRET_TOKEN')

    if secret_token != expected_token:
        return 'Invalid token', 401

    handle_webhook(request.json)
    return 'OK', 200
```

## Webhook Payload Format

### Common Fields

All webhook payloads include these fields:

```typescript
{
    "event": "message.new",           // Event type
    "timestamp": "2024-01-05T11:30:00.000Z",  // ISO 8601 timestamp
    "bot": {
        "id": "694b67f394ee87bd045c2ad1",
        "username": "Bot11",
        "name": "My Bot"
    },
    "room": {                          // Present for room-related events
        "id": "GENERAL",
        "name": "general",
        "type": "c"
    },
    "message": {                      // Present for message events
        "id": "msg123",
        "text": "Hello world!",
        "userId": "user123",
        "username": "john",
        "createdAt": "2024-01-05T11:30:00.000Z"
    },
    "user": {                         // Present for user events
        "id": "user123",
        "username": "john",
        "name": "John Doe"
    }
}
```

### Event Types

**Message Events**

| Event | Description | Included Fields |
|-------|-------------|-----------------|
| `message.new` | New message posted | `bot`, `room`, `message` |
| `message.edited` | Message edited | `bot`, `room`, `message` |
| `message.deleted` | Message deleted | `bot`, `room`, `message` |
| `message.mention` | Message with @mentions | `bot`, `form`, `message` |

**Room Events**

| Event | Description | Included Fields |
|-------|-------------|-----------------|
| `room.joined` | Bot joined a room | `bot`, `room` |
| `room.left` | Bot left a room | `bot`, `room` |

**User Events**

| Event | Description | Included Fields |
|-------|-------------|-----------------|
| `user.joined` | User joined a room | `bot`, `room`, `user` |
| `user.left` | User left a room | `bot`, `room`, `user` |

**List Item Events**

| Event | Description | Top-level IDs | Included Fields |
|-------|-------------|---------------|-----------------|
| `list.item.created` | List item created | `listId`, `itemId` | `bot`, `room`, `item`, `list`, `stage`, `actor` |
| `list.item.deleted` | List item deleted | `listId`, `itemId` | `bot`, `room`, `item`, `list`, `stage?`, `actor` |
| `list.item.stage_changed` | List item moved to different stage | `listId`, `itemId` | `bot`, `room`, `item`, `list`, `newStage`, `previousStage?`, `actor` |
| `list.item.attributes_changed` | List item attributes changed | `listId`, `itemId`, `changedFieldIds?` | `bot`, `room`, `item`, `list`, `stage`, `changedFields`, `actor` |

**File Events**

| Event | Description | Top-level IDs | Included Fields |
|-------|-------------|---------------|-----------------|
| `file.created` | File uploaded | `fileId` | `bot`, `room`, `file`, `actor` |
| `file.updated` | File renamed/moved | `fileId` | `bot`, `room`, `file`, `changes`, `actor` |
| `file.deleted` | File deleted | `fileId` | `bot`, `room`, `file`, `actor` |

**Folder Events**

| Event | Description | Top-level IDs | Included Fields |
|-------|-------------|---------------|-----------------|
| `folder.created` | Folder created | `folderId` | `bot`, `room`, `folder`, `actor` |
| `folder.deleted` | Folder deleted | `folderId` | `bot`, `room`, `folder`, `actor` |
| `folder.renamed` | Folder renamed | `folderId` | `bot`, `room`, `folder`, `previousName`, `actor` |

### Payload Examples

See [Detailed Payload Examples](./bot-webhooks/webhook-payload-examples.md) for comprehensive examples of all event types including:
- Message events (new, edited, deleted, mention, inline buttons)
- List item events (created, deleted, stage changed, attributes changed)
- File events (created, updated, deleted)
- Folder events (created, deleted, renamed)

#### Key Notes

**message.mention**: Uses `form` object (with `roomSessionKey` and `privosEndpointUrl`) instead of standard `room` object. Message uses raw field names (`_id`, `rid`, `msg`) with `mentions` array.

**Inline Button Clicks**: When a button with `bot-event` action is clicked, webhook receives `message.mention` event with `metadata` in the `form` object for identifying the action.

**List/File/Folder Events**: Include `actor` field showing who triggered the event. File events use `folderPath` (display path, not internal MinIO path) for security.

## Sending Messages Back

To send messages back to the room, use the Bot API endpoint:

### POST /api/v1/bot/sendMessage

**Headers:**
```
Authorization: Bearer <your-bot-token>
Content-Type: application/json
```

**Request Body:**

```json
{
    "roomId": "GENERAL",
    "text": "Hello from webhook handler!"
}
```

**With optional parameters:**

```json
{
    "roomId": "GENERAL",
    "text": "Hello from webhook handler!",
    "reply_to_message_id": "msg123",  // Reply to specific message
    "inlineKeyboard": {               // Optional: Inline keyboard buttons
        "type": "actions",            // Type: actions, single_select, multi_select
        "buttons": [
            {
                "id": "btn_approve",
                "text": "Approve",
                "action": "bot-event",
                "actionData": "{\"event\": \"message.mention\", \"botUserId\": \"bot123\", \"metadata\": {\"action\": \"approve\", \"requestId\": \"req_001\"}}"
            },
            {
                "id": "btn_deny",
                "text": "Deny",
                "action": "bot-event",
                "actionData": "{\"event\": \"message.mention\", \"botUserId\": \"bot123\", \"metadata\": {\"action\": \"deny\", \"requestId\": \"req_001\"}}"
            }
        ]
    }
}
```

**cURL Example:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/sendMessage \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -H "Content-Type: application/json" \
  -d '{
    "roomId": "GENERAL",
    "text": "Hello from webhook!"
  }'
```

**Node.js Example:**

```javascript
async function sendMessage(roomId, text, options = {}) {
    const response = await fetch('https://your-privos-chat.com/api/v1/bot/sendMessage', {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${process.env.BOT_TOKEN}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            roomId,
            text,
            ...options
        })
    });

    if (!response.ok) {
        throw new Error(`Failed to send message: ${response.statusText}`);
    }

    return response.json();
}

// Usage in webhook handler
async function handleWebhook(payload) {
    if (payload.event === 'message.new') {
        const message = payload.message.text;

        // Process message and get response
        const response = await processMessage(message);

        // Send response back to room
        await sendMessage(
            payload.room.id,
            response,
            { reply_to_message_id: payload.message.id }  // Reply to message
        );
    }
}
```

**Python Example:**

```python
import requests
import os

BOT_TOKEN = os.environ.get('BOT_TOKEN')
API_URL = 'https://your-privos-chat.com/api/v1/bot/sendMessage'

def send_message(room_id, text, reply_to_message_id=None):
    """Send a message to a room"""
    headers = {
        'Authorization': f'Bearer {BOT_TOKEN}',
        'Content-Type': 'application/json'
    }

    payload = {
        'roomId': room_id,
        'text': text
    }

    if reply_to_message_id:
        payload['reply_to_message_id'] = reply_to_message_id

    response = requests.post(API_URL, json=payload, headers=headers)
    response.raise_for_status()
    return response.json()

def handle_webhook(payload):
    if payload['event'] == 'message.new':
        message_text = payload['message']['text']
        response_text = f"You said: {message_text}"

        send_message(
            payload['room']['id'],
            response_text,
            reply_to_message_id=payload['message']['id']
        )
```

### POST /api/v1/bot/sendPhoto

Send a photo to a room.

**Headers:**
```
Authorization: Bearer <your-bot-token>
Content-Type: multipart/form-data
```

**Request Body (multipart/form-data):**
```
roomId: GENERAL
photo: <file or URL>
caption: Optional photo caption
reply_to_message_id: Optional message ID to reply to
```

**cURL Example:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/sendPhoto \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -F "roomId=GENERAL" \
  -F "photo=@/path/to/photo.jpg" \
  -F "caption=Check out this photo!"
```

**With reply_to_message_id:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/sendPhoto \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -F "roomId=GENERAL" \
  -F "photo=@/path/to/photo.jpg" \
  -F "caption=Check out this photo!" \
  -F "reply_to_message_id=msg123"
```

**Node.js Example:**

```javascript
const FormData = require('form-data');
const fs = require('fs');
const fetch = require('node-fetch');

async function sendPhoto(roomId, photoPath, caption = '') {
    const form = new FormData();
    form.append('roomId', roomId);
    form.append('photo', fs.createReadStream(photoPath));
    if (caption) form.append('caption', caption);

    const response = await fetch('https://your-privos-chat.com/api/v1/bot/sendPhoto', {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${process.env.BOT_TOKEN}`,
            ...form.getHeaders()
        },
        body: form
    });

    return response.json();
}
```

### POST /api/v1/bot/sendAudio

Send an audio file to a room (only .mp3 and .m4a formats supported).

**Headers:**
```
Authorization: Bearer <your-bot-token>
Content-Type: multipart/form-data
```

**Request Body (multipart/form-data):**
```
roomId: GENERAL
audio: <file> (.mp3 or .m4a)
reply_to_message_id: Optional message ID to reply to
```

**cURL Example:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/sendAudio \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -F "roomId=GENERAL" \
  -F "audio=@/path/to/audio.mp3"
```

**With reply_to_message_id:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/sendAudio \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -F "roomId=GENERAL" \
  -F "audio=@/path/to/audio.mp3" \
  -F "reply_to_message_id=msg123"
```

### POST /api/v1/bot/sendDocument

Send a document/file to a room (max 50MB).

**Headers:**
```
Authorization: Bearer <your-bot-token>
Content-Type: multipart/form-data
```

**Request Body (multipart/form-data):**
```
roomId: GENERAL
document: <file> (max 50MB)
reply_to_message_id: Optional message ID to reply to
```

**cURL Example:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/sendDocument \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -F "roomId=GENERAL" \
  -F "document=@/path/to/document.pdf"
```

**With reply_to_message_id:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/sendDocument \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -F "roomId=GENERAL" \
  -F "document=@/path/to/document.pdf" \
  -F "reply_to_message_id=msg123"
```

### POST /api/v1/bot/sendVideo

Send a video to a room.

**Headers:**
```
Authorization: Bearer <your-bot-token>
Content-Type: multipart/form-data
```

**Request Body (multipart/form-data):**
```
roomId: GENERAL
video: <file>
reply_to_message_id: Optional message ID to reply to
```

**cURL Example:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/sendVideo \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -F "roomId=GENERAL" \
  -F "video=@/path/to/video.mp4"
```

**With reply_to_message_id:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/sendVideo \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -F "roomId=GENERAL" \
  -F "video=@/path/to/video.mp4" \
  -F "reply_to_message_id=msg123"
```

### POST /api/v1/bot/sendVoice

Send a voice message to a room (.ogg, .oga, .opus, .mp3, .webm formats supported).

**Headers:**
```
Authorization: Bearer <your-bot-token>
Content-Type: multipart/form-data
```

**Request Body (multipart/form-data):**
```
roomId: GENERAL
voice: <file> (.ogg, .oga, .opus, .mp3, or .webm)
reply_to_message_id: Optional message ID to reply to
```

**cURL Example:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/sendVoice \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -F "roomId=GENERAL" \
  -F "voice=@/path/to/voice.ogg"
```

**With reply_to_message_id:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/sendVoice \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -F "roomId=GENERAL" \
  -F "voice=@/path/to/voice.ogg" \
  -F "reply_to_message_id=msg123"
```

### POST /api/v1/bot/editMessage

Edit a previously sent message.

**Headers:**
```
Authorization: Bearer <your-bot-token>
Content-Type: application/json
```

**Request Body:**
```json
{
    "messageId": "msg123",
    "text": "Updated message text"
}
```

**cURL Example:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/editMessage \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "msg123",
    "text": "Updated text"
  }'
```

### POST /api/v1/bot/deleteMessage

Delete a previously sent message.

**Headers:**
```
Authorization: Bearer <your-bot-token>
Content-Type: application/json
```

**Request Body:**
```json
{
    "messageId": "msg123"
}
```

**cURL Example:**

```bash
curl -X POST https://your-privos-chat.com/api/v1/bot/deleteMessage \
  -H "Authorization: Bearer privos_694b67f394ee87bd045c2ad1_a1b2c3d4e5f6" \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "msg123"
  }'
```

## Inline Keyboard Callback Handling

### How Inline Button Clicks Work

When a user clicks an inline button with a `bot-event` action, the system:

1. Client sends button click to `POST /api/v1/chat.keyboardCallback`
2. Server parses the button's `actionData` (JSON string)
3. Server delivers webhook to the bot using the specified event (e.g., `message.mention`)
4. Bot receives the webhook with `metadata` in the `form` field

### Distinguishing Button Clicks in Your Webhook Handler

The simplest way to detect an inline button click is to check for `metadata` in the `form` object:


**Python Example:**
```python
def handle_webhook(payload):
    if payload['event'] == 'message.mention':
        # Check if this is an inline button click
        metadata = payload.get('form', {}).get('metadata')

        if metadata:
            # This is an inline button click
            action = metadata.get('action')
            request_id = metadata.get('requestId')

            print(f"Button clicked: {action} for request {request_id}")

            if action == 'approve':
                approve_request(request_id)
            elif action == 'deny':
                deny_request(request_id)
        else:
            # This is a regular @mention
            handle_mention(payload['message']['mentions'])
```

### Button Action Data Format

The `actionData` field in a bot-event button must be a JSON string containing:

```json
{
    "event": "message.mention",           // Event type to trigger
    "botUserId": "bot_id_123",           // Target bot's user ID
    "metadata": {                         // Custom metadata (passed in webhook)
        "action": "approve",
        "requestId": "req_001",
        "userId": "user_123"
    }
}
```

All fields in `metadata` are passed directly to the webhook's `form.metadata` object.

## Error Handling

Your webhook endpoint should:

1. **Return 2xx status codes** on success (recommended: `200 OK`)
2. **Return 401** for authentication failures
3. **Return 4xx/5xx** for processing errors (webhook will be retried)
4. **Respond quickly** (within 30 seconds is recommended)

## Webhook Retries

If your webhook returns a non-2xx status code or times out:

- The webhook will be retried with **exponential backoff**
- Initial delay: **30 seconds** (configurable)
- Max delay: **10 minutes** (configurable)
- Max attempts: **24** (configurable)

## Best Practices

1. **Always verify signatures** before processing webhook data
2. **Use idempotency keys** if your processing is not idempotent
3. **Return 200 OK quickly** - do heavy processing asynchronously
4. **Log all webhook events** for debugging
5. **Monitor webhook health** using the `lastTriggeredAt` and `failureCount` fields
6. **Handle network timeouts gracefully** - the queue will retry failed deliveries

## Testing Your Webhook

Use webhook testing tools like:
- **webhook.site** - Quick testing (no auth required)
- **ngrok** - Expose local server for testing
- **RequestBin** - Inspect webhook requests

## Example: Simple Webhook Server (Node.js)

```javascript
const express = require('express');

const app = express();
const PORT = process.env.PORT || 3000;
const WEBHOOK_SECRET_TOKEN = process.env.WEBHOOK_SECRET_TOKEN;
const BOT_TOKEN = process.env.BOT_TOKEN;

// Parse JSON body
app.use(express.json());

app.post('/webhook', (req, res) => {
    try {
        // Verify secret token
        const secretToken = req.headers['x-privos-bot-api-secret-token'];

        if (secretToken !== WEBHOOK_SECRET_TOKEN) {
            console.error('Invalid token');
            return res.status(401).send('Invalid token');
        }

        // Process webhook
        const payload = req.body;
        console.log('Received webhook:', payload.event);

        handleWebhookEvent(payload);

        res.status(200).send('OK');
    } catch (error) {
        console.error('Webhook error:', error);
        res.status(500).send('Internal server error');
    }
});

async function handleWebhookEvent(payload) {
    switch (payload.event) {
        case 'message.new':
            await handleNewMessage(payload);
            break;
        case 'message.edited':
            await handleEditedMessage(payload);
            break;
        // Handle other events...
    }
}

async function handleNewMessage(payload) {
    const { room, message } = payload;
    const response = `Echo: ${message.text}`;

    // Send response back to Privos
    await fetch('https://your-privos-chat.com/api/v1/bot/sendMessage', {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${BOT_TOKEN}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            roomId: room.id,
            text: response,
            userId: message.userId
        })
    });
}

app.listen(PORT, () => {
    console.log(`Webhook server listening on port ${PORT}`);
});
```

## Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `WEBHOOK_SECRET_TOKEN` | Your webhook secret token for authentication | `my-random-token` |
| `BOT_TOKEN` | Your bot API token for sending messages | `privos_694b67f3...` |
| `API_URL` | Privos Chat API base URL | `https://your-privos-chat.com` |

## Security Considerations

1. **Never share your webhook secret token** - it's used to authenticate webhook requests
2. **Always verify the token** - even for "trusted" sources
3. **Use HTTPS** - protect webhook data in transit
4. **Rate limit** - protect against webhook floods
5. **Validate payloads** - check required fields exist
6. **Sanitize user input** - prevent XSS when echoing messages
7. **Keep tokens secure** - use environment variables, never commit them

## Troubleshooting

### Webhook not received?

1. Check the webhook is **active** in the bot configuration
2. Verify the webhook **URL is correct** and accessible
3. Check the **event types** are enabled (e.g., `message.new`)
4. Ensure the bot is **in the room** where the event occurred
5. Verify **roomIds** filter isn't excluding the room

### Token verification failing?

1. Check you're using the correct **secret token** from bot configuration
2. Verify the token is being sent in the `X-Privos-Bot-Api-Secret-Token` header
3. Ensure the token matches exactly (case-sensitive)

### Messages not sending back?

1. Verify the **bot token** is valid
2. Check the **roomId** is correct
3. Ensure your bot has **permission** to post in the room
4. Check the **API URL** is correct

## Additional Resources

- Bot API Documentation: See `/docs/BOT_API_FEATURE.md`
- Webhook Queue Architecture: See `/docs/WEBHOOK_QUEUE_ARCHITECTURE.md`
- Example implementations: See `/examples/webhook-handlers/`
