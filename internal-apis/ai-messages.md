# AI Messages API (Client-Facing)

Client-facing API for AI chat messaging, session management, and context-aware session resume.

## Base URL

```
/api/v1/ai-messages
```

## Authentication

All endpoints require standard Rocket.Chat user authentication (JWT token in `Authorization: Bearer` header).

---

## Endpoints

### Send Message

```http
POST /v1/ai-messages.send
```

Send a message in an AI chat session, updating attached contexts automatically.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `sessionId` | string | Yes | AI chat session ID |
| `message` | string | Yes | Message text |
| `attachedContexts` | array | No | Array of attached contexts to track |
| `context` | string | No | Optional context for AI instruction |
| `referenceAnswer` | string | No | Optional reference answer |
| `contextPrefix` | string | No | Prefix for inserted context |

**Context Object:**

```typescript
{
  type: string;      // e.g., "item", "document", "list"
  name: string;      // Human-readable name
  id?: string;       // Optional ID
}
```

**Request:**

```json
{
  "roomId": "ROOM_123",
  "sessionId": "SESSION_456",
  "message": "Can you help with project planning?",
  "attachedContexts": [
    { "type": "item", "name": "Q2 Planning Task", "id": "ITEM_456" },
    { "type": "document", "name": "Project Brief", "id": "DOC_123" }
  ]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": {
      "_id": "MSG_ID",
      "roomId": "ROOM_123",
      "sessionId": "SESSION_456",
      "text": "Can you help with project planning?",
      "userId": "USER_ID",
      "timestamp": "2024-03-24T12:00:00Z",
      "attachedContexts": [
        { "type": "item", "name": "Q2 Planning Task", "id": "ITEM_456" },
        { "type": "document", "name": "Project Brief", "id": "DOC_123" }
      ]
    }
  }
}
```

**Behavior:**
- Creates new session message
- Updates session's `attachedContexts` with provided contexts
- Emits real-time message event to subscribed clients
- AI agent receives message with context information

---

### Create Session

```http
POST /v1/ai-messages.createSession
```

Create a new AI chat session with optional entity context.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `entityType` | string | No | Entity type (e.g., "item", "list", "file") |
| `entityId` | string | No | Entity ID for primary context |
| `entityName` | string | No | Human-readable entity name |
| `itemId` | string | No | Optional secondary item context |
| `sessionTitle` | string | No | Session title (for display) |

**Request:**

```json
{
  "roomId": "ROOM_123",
  "entityType": "item",
  "entityId": "ITEM_456",
  "entityName": "Q2 Planning Task",
  "itemId": "ITEM_789",
  "sessionTitle": "Planning Discussion"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "session": {
      "_id": "SESSION_456",
      "roomId": "ROOM_123",
      "entityType": "item",
      "entityId": "ITEM_456",
      "entityName": "Q2 Planning Task",
      "itemId": "ITEM_789",
      "sessionTitle": "Planning Discussion",
      "messageCount": 0,
      "attachedContexts": [],
      "participants": [],
      "createdAt": "2024-03-24T12:00:00Z"
    }
  }
}
```

---

### Get Session

```http
GET /v1/ai-messages.getSession
```

Retrieve a specific AI chat session with its data.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionId` | string | Yes | Session ID |
| `roomId` | string | Yes | Room ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "session": {
      "_id": "SESSION_456",
      "roomId": "ROOM_123",
      "entityType": "item",
      "entityId": "ITEM_456",
      "entityName": "Q2 Planning Task",
      "itemId": "ITEM_789",
      "sessionTitle": "Planning Discussion",
      "messageCount": 5,
      "lastMessageAt": "2024-03-24T12:30:00Z",
      "attachedContexts": [
        { "type": "item", "name": "Q2 Planning Task", "id": "ITEM_456" },
        { "type": "document", "name": "Project Brief", "id": "DOC_123" }
      ],
      "participants": [...],
      "createdAt": "2024-03-24T12:00:00Z"
    }
  }
}
```

---

### List Sessions (with Context Filtering)

```http
GET /v1/ai-messages.sessions
```

Retrieve all sessions for a room, optionally filtered by entity context.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `entityType` | string | No | Filter by entity type (e.g., "item", "list", "file") |
| `entityId` | string | No | Filter by entity ID |
| `itemId` | string | No | Filter by secondary item context |
| `search` | string | No | Search session titles |
| `limit` | number | No | Limit results (default: 50) |
| `offset` | number | No | Pagination offset (default: 0) |

**Response:**

```json
{
  "success": true,
  "data": {
    "sessions": [
      {
        "_id": "SESSION_456",
        "roomId": "ROOM_123",
        "entityType": "item",
        "entityId": "ITEM_456",
        "entityName": "Q2 Planning Task",
        "sessionTitle": "Planning Discussion",
        "messageCount": 5,
        "lastMessageAt": "2024-03-24T12:30:00Z",
        "attachedContexts": [
          { "type": "item", "name": "Q2 Planning Task", "id": "ITEM_456" }
        ],
        "createdAt": "2024-03-24T12:00:00Z"
      }
    ],
    "count": 1,
    "total": 10,
    "offset": 0
  }
}
```

**Examples:**

```bash
# Get all sessions
GET /v1/ai-messages.sessions?roomId=ROOM_123

# Get sessions for specific item
GET /v1/ai-messages.sessions?roomId=ROOM_123&entityType=item&entityId=ITEM_456

# Get sessions for file context
GET /v1/ai-messages.sessions?roomId=ROOM_123&entityType=file&entityId=FOLDER_123

# Search sessions
GET /v1/ai-messages.sessions?roomId=ROOM_123&search=planning
```

---

### Get Messages

```http
GET /v1/ai-messages.list
```

Retrieve messages from an AI chat session.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionId` | string | Yes | Session ID |
| `roomId` | string | Yes | Room ID |
| `limit` | number | No | Limit results (default: 50) |
| `offset` | number | No | Pagination offset (default: 0) |

**Response:**

```json
{
  "success": true,
  "data": {
    "messages": [
      {
        "_id": "MSG_1",
        "sessionId": "SESSION_456",
        "text": "Help me with planning",
        "userId": "USER_123",
        "timestamp": "2024-03-24T12:00:00Z",
        "isUser": true
      },
      {
        "_id": "MSG_2",
        "sessionId": "SESSION_456",
        "text": "I'll help you create a plan...",
        "isUser": false,
        "timestamp": "2024-03-24T12:01:00Z"
      }
    ],
    "count": 2,
    "total": 8,
    "offset": 0
  }
}
```

---

### Clear Session Entity Context

```http
POST /v1/ai-messages.clearSessionEntityContext
```

Remove entity context from a session (unlinks from specific entity).

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionId` | string | Yes | Session ID |
| `roomId` | string | Yes | Room ID |

**Request:**

```json
{
  "sessionId": "SESSION_456",
  "roomId": "ROOM_123"
}
```

**Response:**

```json
{
  "success": true
}
```

**Behavior:**
- Removes `entityType`, `entityId`, and `itemId` from session
- Session persists but is no longer associated with entity
- Session can still be accessed from session list (unfiltered)

---

### Update Question Answers

```http
POST /v1/ai-messages.updateQuestionAnswers
```

Submit answers to pending questions in AI chat flow.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionId` | string | Yes | Session ID |
| `roomId` | string | Yes | Room ID |
| `answers` | object | Yes | Answers to questions |

**Request:**

```json
{
  "sessionId": "SESSION_456",
  "roomId": "ROOM_123",
  "answers": {
    "timeline": {
      "header": "Timeline",
      "answer": "Q2 2024",
      "isCustom": false
    },
    "budget": {
      "header": "Budget",
      "answer": "$50,000",
      "isCustom": true
    }
  }
}
```

**Response:**

```json
{
  "success": true
}
```

---

## Error Responses

| Error Code | Description |
|------------|-------------|
| `error-invalid-params` | Missing or invalid parameters |
| `error-unauthorized` | Authentication required |
| `error-forbidden` | No access to room/session |
| `error-session-not-found` | Session not found |
| `error-room-not-found` | Room not found |

**Error Response Example:**

```json
{
  "success": false,
  "error": "error-session-not-found",
  "message": "AI chat session not found"
}
```

---

## Real-Time Events

### Message Events

Subscribe to session message stream:

```typescript
RocketChat.subscribe('agent-chat', sessionId, (data: {
  sessionId: string;
  message: {
    _id: string;
    text: string;
    timestamp: Date;
    isUser: boolean;
    attachedContexts?: Array<{ type: string; name: string; id?: string }>;
  };
  documentUpdated?: boolean;
  isModified?: boolean;
}) => {
  // Handle new message
});
```

---

## Context-Aware Session Resume Workflow

When user opens AI chat:

1. Resolve current entity context (from page/route)
2. Call `GET /v1/ai-messages.sessions?entityType=...&entityId=...`
3. If sessions exist:
   - Load most recent session
   - Restore message history
   - Display attached context chips
4. If no sessions:
   - Create new session with `POST /v1/ai-messages.createSession`
   - Show empty chat
5. While chat is open:
   - On entity navigation: keep session, show insert context button
   - On each message: update attachedContexts with current contexts
   - On session switch: restore that session's saved contexts
6. On chat close: reset to initial state

---

## Session Storage

Client stores session state using `aiChatSessionStorage` (v3 format):

```typescript
{
  v: 3,                        // Storage version
  entityContext: {
    type: 'item' | 'list' | 'file';
    id: string;
  },
  session: {
    id: string;
    title: string;
    messageCount: number;
  },
  timestamp: number;
}
```

Storage key: `{roomId}:ai-chat:${entityContext.type}:${entityContext.id}`

---

## Related Resources

- [Agent Chat Session API (Internal)](./agent-chat-session.md) - Server-side session management
- [Room-Scoped AI Chat Sessions API](../room-scoped-apis/ai-chat-sessions.md) - Room-scoped operations
- [Agent Chat API](./agent-chat.md) - Bot message sending
