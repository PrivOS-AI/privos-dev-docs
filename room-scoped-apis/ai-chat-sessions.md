# Room-Scoped AI Chat Sessions API

## Overview

Manage AI chat sessions with context-aware resume capabilities and canvas artifacts within a specific room. Sessions can be filtered by entity context, and attached contexts persist across messages to enable intelligent session recovery.

**Base Path:** `/api/v1/internal/rooms/:roomId/ai-chat-sessions`

## Authentication

All endpoints require:
- `x-api-key`: Internal API key
- `Authorization: Bearer <JWT_TOKEN>`: Room-specific JWT token

---

## Endpoints

### Get All Sessions (with Context Filtering)

```http
GET /api/v1/internal/rooms/:roomId/ai-chat-sessions
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entityType` | string | No | Filter by entity type (e.g., "item", "list", "file") |
| `entityId` | string | No | Filter by entity ID |
| `itemId` | string | No | Filter by secondary item context |
| `search` | string | No | Search session titles |
| `limit` | number | No | Limit results (default: 50) |
| `skip` | number | No | Skip results for pagination (default: 0) |

**Description:** Get all AI chat sessions for the room, optionally filtered by entity context.

**Response:**

```json
{
  "success": true,
  "data": {
    "sessions": [
      {
        "_id": "SESSION_ID",
        "roomId": "ROOM_ID",
        "sessionTitle": "Q2 Planning",
        "entityType": "item",
        "entityId": "ITEM_456",
        "entityName": "Q2 Planning Task",
        "itemId": "ITEM_789",
        "messageCount": 5,
        "lastMessageAt": "2024-03-24T11:30:00Z",
        "attachedContexts": [
          { "type": "item", "name": "Q2 Planning Task", "id": "ITEM_456" },
          { "type": "document", "name": "Project Brief", "id": "DOC_123" }
        ],
        "participants": [...],
        "createdAt": "2024-03-24T10:00:00Z"
      }
    ],
    "count": 1,
    "total": 15,
    "offset": 0
  }
}
```

**Examples:**

```bash
# Get all sessions in room
GET /api/v1/internal/rooms/ROOM_ID/ai-chat-sessions

# Get sessions for a specific item
GET /api/v1/internal/rooms/ROOM_ID/ai-chat-sessions?entityType=item&entityId=ITEM_456

# Get sessions for item with secondary context
GET /api/v1/internal/rooms/ROOM_ID/ai-chat-sessions?entityType=item&entityId=ITEM_456&itemId=ITEM_789

# Get sessions for file browser context
GET /api/v1/internal/rooms/ROOM_ID/ai-chat-sessions?entityType=file&entityId=FOLDER_123

# Search sessions by title
GET /api/v1/internal/rooms/ROOM_ID/ai-chat-sessions?search=quarterly
```

---

### Get Latest Session by Entity (Context-Aware Resume)

```http
GET /api/v1/internal/rooms/:roomId/ai-chat-sessions/latest-by-entity
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entityType` | string | Yes | Entity type (e.g., "item", "list", "file") |
| `entityId` | string | Yes | Entity ID |
| `itemId` | string | No | Optional secondary item context |

**Description:** Get the most recent session for a specific entity context. Used for context-aware session auto-selection when opening AI chat.

**Response:**

```json
{
  "success": true,
  "data": {
    "session": {
      "_id": "SESSION_789",
      "roomId": "ROOM_ID",
      "entityType": "item",
      "entityId": "ITEM_456",
      "entityName": "Q2 Planning Task",
      "itemId": "ITEM_789",
      "messageCount": 8,
      "lastMessageAt": "2024-03-24T12:00:00Z",
      "attachedContexts": [
        { "type": "item", "name": "Q2 Planning Task", "id": "ITEM_456" },
        { "type": "document", "name": "Project Brief", "id": "DOC_123" }
      ],
      "participants": [...],
      "createdAt": "2024-03-24T10:00:00Z"
    }
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/ai-chat-sessions/latest-by-entity?entityType=item&entityId=ITEM_456" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Clear Session Entity Context

```http
POST /api/v1/internal/rooms/:roomId/ai-chat-sessions/:sessionId/clear-entity-context
```

**Description:** Remove entity context from a session (unlinks the session from a specific entity). The session persists but is no longer associated with that entity.

**Response:**

```json
{
  "success": true
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/ai-chat-sessions/SESSION_ID/clear-entity-context" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Update Attached Contexts

```http
POST /api/v1/internal/rooms/:roomId/ai-chat-sessions/:sessionId/update-contexts
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `contexts` | array | Yes | Array of attached contexts |

**Context Object:**

```typescript
{
  type: string;      // e.g., "item", "document", "list"
  name: string;      // Human-readable name
  id?: string;       // Optional ID
}
```

**Description:** Update the list of attached contexts for a session. Called automatically on each message send to track all contexts involved in the conversation.

**Request:**

```json
{
  "contexts": [
    { "type": "item", "name": "Q2 Planning Task", "id": "ITEM_456" },
    { "type": "document", "name": "Project Brief", "id": "DOC_123" },
    { "type": "list", "name": "Strategic Planning", "id": "LIST_789" }
  ]
}
```

**Response:**

```json
{
  "success": true
}
```

---

### Get Canvas Artifacts

```http
GET /api/v1/internal/rooms/:roomId/ai-chat-sessions/artifacts
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionId` | string | Yes | AI chat session ID |

**Description:** Get canvas artifacts (markdown and HTML) from an AI chat session.

**Response:**

```json
{
  "success": true,
  "data": {
    "artifacts": {
      "markdownCanvas": {
        "_id": "CANVAS_ID",
        "title": "Project Plan",
        "content": "# Project Plan\n\n## Overview\n...",
        "description": "Generated project plan",
        "canvasType": "markdown",
        "versions": [
          {
            "version": 1,
            "content": "Initial content...",
            "createdAt": "2024-01-01T00:00:00Z"
          }
        ],
        "currentVersion": 1,
        "createdAt": "2024-01-01T00:00:00Z",
        "updatedAt": "2024-01-02T00:00:00Z"
      },
      "htmlCanvas": {
        "_id": "CANVAS_ID_2",
        "title": "Landing Page",
        "content": "<!DOCTYPE html>...",
        "description": "Generated landing page",
        "canvasType": "html",
        "versions": [...],
        "currentVersion": 1,
        "createdAt": "2024-01-01T00:00:00Z",
        "updatedAt": "2024-01-02T00:00:00Z"
      }
    }
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/ai-chat-sessions/artifacts?sessionId=SESSION_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Get AI Chat Session

```http
GET /api/v1/internal/rooms/:roomId/ai-chat-sessions/:sessionId
```

**Description:** Get a specific AI chat session with all its data.

**Response:**

```json
{
  "success": true,
  "data": {
    "session": {
      "_id": "SESSION_ID",
      "roomId": "ROOM_ID",
      "entityType": "item",
      "entityId": "ITEM_456",
      "entityName": "Q2 Planning Task",
      "itemId": "ITEM_789",
      "sessionTitle": "Project Planning Session",
      "messageCount": 12,
      "lastMessageAt": "2024-03-24T12:00:00Z",
      "attachedContexts": [
        { "type": "item", "name": "Q2 Planning Task", "id": "ITEM_456" },
        { "type": "document", "name": "Project Brief", "id": "DOC_123" }
      ],
      "participants": [
        {
          "userId": "USER_ID",
          "username": "john.doe",
          "name": "John Doe",
          "firstMessageAt": "2024-03-24T10:00:00Z",
          "lastMessageAt": "2024-03-24T12:00:00Z",
          "messageCount": 5
        }
      ],
      "flowChatId": "flow_abc123",
      "flowSessionId": "flow_session_xyz",
      "markdownCanvas": { ... },
      "htmlCanvas": { ... },
      "createdAt": "2024-03-24T10:00:00Z",
      "createdBy": {
        "_id": "USER_ID",
        "username": "john.doe",
        "name": "John Doe"
      }
    }
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/ai-chat-sessions/SESSION_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Delete AI Chat Session

```http
DELETE /api/v1/internal/rooms/:roomId/ai-chat-sessions/:sessionId
```

**Description:** Delete an AI chat session and its associated data.

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "AI chat session deleted successfully",
    "deleted": true
  }
}
```

**Example:**

```bash
curl -X DELETE "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/ai-chat-sessions/SESSION_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Update Canvas Content

```http
POST /api/v1/internal/rooms/:roomId/ai-chat-sessions/canvas-update
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionId` | string | Yes | AI chat session ID |
| `canvasType` | string | Yes | Canvas type: `markdown` or `html` |
| `canvasContent` | string | Yes | Canvas content |
| `title` | string | No | Canvas title (optional, uses default if not provided) |

**Description:** Update canvas content for a session. Creates a new version with each update.

**Response:**

```json
{
  "success": true,
  "data": {
    "success": true,
    "canvas": {
      "_id": "CANVAS_ID",
      "title": "Updated Project Plan",
      "content": "# Updated Project Plan\n\n## Changes\n...",
      "description": "",
      "canvasType": "markdown",
      "versions": [
        {
          "version": 1,
          "content": "Original content...",
          "createdAt": "2024-01-01T00:00:00Z",
          "createdBy": {
            "_id": "ai-assistant",
            "username": "ai-assistant",
            "name": "AI Assistant"
          }
        },
        {
          "version": 2,
          "content": "Updated content...",
          "createdAt": "2024-01-02T12:00:00Z",
          "createdBy": {
            "_id": "ai-assistant",
            "username": "ai-assistant",
            "name": "AI Assistant"
          }
        }
      ],
      "currentVersion": 2,
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-02T12:00:00Z",
      "updatedBy": {
        "_id": "ai-assistant",
        "username": "ai-assistant",
        "name": "AI Assistant"
      }
    }
  }
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/ai-chat-sessions/canvas-update" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "SESSION_ID",
    "canvasType": "markdown",
    "canvasContent": "# Updated Project Plan\n\n## Phase 1\n\n...",
    "title": "Updated Project Plan"
  }'
```

---

## Error Codes

| Error Code | Description |
|------------|-------------|
| `error-session-not-found` | AI chat session not found in this room |
| `error-invalid-canvas-type` | Invalid canvas type (must be `markdown` or `html`) |
| `error-forbidden` | Session doesn't belong to this room |
| `error-invalid-params` | Missing or invalid parameters |

---

## Canvas Types

### Markdown Canvas
- **Type:** `markdown`
- **Content Format:** Markdown text
- **Use Case:** Documentation, project plans, notes
- **Default Title:** "Canvas"

### HTML Canvas
- **Type:** `html`
- **Content Format:** HTML markup
- **Use Case:** Landing pages, UI mockups, web content
- **Default Title:** "Landing Page"

---

## Canvas Versioning

Each canvas maintains a version history:

```typescript
{
  versions: [
    {
      version: 1,
      content: "Initial content...",
      createdAt: Date,
      createdBy: {
        _id: string,
        username: string,
        name: string
      }
    },
    {
      version: 2,
      content: "Updated content...",
      createdAt: Date,
      createdBy: { ... }
    }
  ],
  currentVersion: 2
}
```

**Versioning Rules:**
- Each update creates a new version
- Versions are immutable once created
- `currentVersion` tracks the active version
- All versions are preserved in history

---

## Context-Aware Session Resume Flow

When a user opens AI chat with an entity context, the following flow occurs:

```
┌─────────────────────┐
│  User opens AI      │
│  chat with entity   │
│  (item/file)        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│ Resolve entity context (e.g., item ID)  │
└──────────┬──────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────┐
│ Call: GET /latest-by-entity?                    │
│        entityType=item&                         │
│        entityId=ITEM_456                        │
└──────────┬──────────────────────────────────────┘
           │
           ▼
       ┌───────────────────────┐
       │ Session found?        │
       └─┬─────────────────┬───┘
         │                 │
     Yes │                 │ No
         │                 │
         ▼                 ▼
    ┌────────────┐  ┌──────────────┐
    │  Load      │  │  Create new  │
    │  session   │  │  session     │
    │  & history │  │  & show      │
    │            │  │  empty chat  │
    └────┬───────┘  └──────┬───────┘
         │                 │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │  Display chat   │
         │  with attached  │
         │  context chips  │
         └─────────────────┘
```

**Behavior Details:**

1. **Session Auto-Selection**: When opening chat with an entity, the latest matching session is automatically selected
2. **Context Chips**: All `attachedContexts` are displayed as chips in the chat header
3. **Navigation Preservation**: If chat stays open during entity navigation, the session persists
4. **Insert Context Button**: While chat is open, users can add new contexts
5. **Message Context Update**: Each message send includes current `attachedContexts`
6. **Session Switching**: Users can switch to different sessions; switching restores that session's saved contexts

---

## Real-time Updates

When canvas content is updated, the server emits a real-time event:

```typescript
AIChatStreamer.emit('canvas-updated', {
  sessionId: string,
  canvasType: 'markdown' | 'html',
  canvas: CanvasObject
});
```

Clients can subscribe to these events to receive real-time updates when canvas content changes.
