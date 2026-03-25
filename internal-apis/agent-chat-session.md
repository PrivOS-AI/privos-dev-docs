# Agent Chat Session API

Agent Chat Session API manages AI chat sessions with context-aware session resume and canvas artifacts (Markdown and HTML).

## Base URL

```
/api/v1/internal/agent-chat-session
```

## Context-Aware Session Resume

Sessions now support attached contexts that enable context-aware session resume. When users navigate between different entities (files, items, lists) while an AI chat is open, the system can:

- Auto-select the most relevant session based on current entity context
- Persist multiple contexts within a single session
- Update contexts as new messages are sent
- Filter sessions by entity type and entity ID

Key fields:
- `entityType`: Type of entity (e.g., "item", "list", "file")
- `entityId`: ID of the primary entity
- `entityName`: Human-readable name of the entity
- `itemId`: Optional secondary item context
- `attachedContexts`: Array of all attached contexts `[{ type: string; name: string; id?: string }]`

## Endpoints

### Create Session with Entity Context

```http
POST /api/v1/internal/agent-chat-session.createSession
```

Create a new AI chat session with optional entity context for context-aware resume.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `flowChatId` | string | Yes | Flowise chat ID |
| `sessionTitle` | string | No | Session title (for display) |
| `entityType` | string | No | Entity type (e.g., "item", "list", "file") |
| `entityId` | string | No | Entity ID for primary context |
| `entityName` | string | No | Human-readable entity name |
| `itemId` | string | No | Optional secondary item context |

**Request:**

```json
{
  "roomId": "ROOM_123",
  "flowChatId": "flow_abc",
  "sessionTitle": "Project Planning",
  "entityType": "item",
  "entityId": "ITEM_456",
  "entityName": "Q2 Planning Task"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "session": {
      "_id": "SESSION_789",
      "roomId": "ROOM_123",
      "flowChatId": "flow_abc",
      "sessionTitle": "Project Planning",
      "entityType": "item",
      "entityId": "ITEM_456",
      "entityName": "Q2 Planning Task",
      "messageCount": 0,
      "participants": [],
      "attachedContexts": [],
      "createdAt": "2024-03-24T10:00:00.000Z"
    }
  }
}
```

---

### Get Latest Session by Entity

```http
GET /api/v1/internal/agent-chat-session.getLatestSessionByEntity
```

Retrieve the most recent session for a specific entity context (for context-aware resume).

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `entityType` | string | Yes | Entity type (e.g., "item", "list", "file") |
| `entityId` | string | Yes | Entity ID |
| `itemId` | string | No | Optional secondary item context |

**Request:**

```bash
GET /api/v1/internal/agent-chat-session.getLatestSessionByEntity?roomId=ROOM_123&entityType=item&entityId=ITEM_456&itemId=ITEM_789
```

**Response:**

```json
{
  "success": true,
  "data": {
    "session": {
      "_id": "SESSION_789",
      "roomId": "ROOM_123",
      "entityType": "item",
      "entityId": "ITEM_456",
      "entityName": "Q2 Planning Task",
      "itemId": "ITEM_789",
      "messageCount": 5,
      "lastMessageAt": "2024-03-24T11:30:00.000Z",
      "attachedContexts": [
        { "type": "item", "name": "Q2 Planning Task", "id": "ITEM_456" },
        { "type": "document", "name": "Project Brief", "id": "DOC_123" }
      ],
      "participants": [...],
      "createdAt": "2024-03-24T10:00:00.000Z"
    }
  }
}
```

---

### Update Attached Contexts

```http
POST /api/v1/internal/agent-chat-session.updateAttachedContexts
```

Update the list of attached contexts for a session (called on each message send).

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionId` | string | Yes | Session ID |
| `contexts` | array | Yes | Array of attached contexts |

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
  "sessionId": "SESSION_789",
  "contexts": [
    { "type": "item", "name": "Q2 Planning Task", "id": "ITEM_456" },
    { "type": "document", "name": "Project Brief", "id": "DOC_123" }
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

### Clear Session Entity Context

```http
POST /api/v1/internal/agent-chat-session.clearSessionEntityContext
```

Remove entity context from a session (unlinks the session from a specific entity).

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionId` | string | Yes | Session ID |

**Request:**

```json
{
  "sessionId": "SESSION_789"
}
```

**Response:**

```json
{
  "success": true
}
```

---

### Get Artifacts

```http
GET /api/v1/internal/agent-chat-session.getArtifacts?entityType=item&entityId=ITEM_ID&itemId=ITEM_ID
```

Retrieve canvas artifacts for a specific entity or item.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entityType` | string | Yes | Entity type (e.g., "item", "list", "room") |
| `entityId` | string | Yes | Entity ID |
| `itemId` | string | No | Item ID (for item-specific sessions) |

**Request Examples:**

```bash
# Get session for an entity
GET /api/v1/internal/agent-chat-session.getArtifacts?entityType=item&entityId=ITEM_123

# Get session for a specific item
GET /api/v1/internal/agent-chat-session.getArtifacts?entityType=item&entityId=ITEM_123&itemId=ITEM_456
```

**Response:**

```json
{
  "success": true,
  "data": {
    "artifacts": {
      "markdownCanvas": {
        "_id": "canvas_1",
        "title": "Project Plan",
        "content": "# Project Plan\n\n## Overview\n...",
        "description": "",
        "canvasType": "markdown",
        "currentVersion": 2,
        "versions": [...],
        "createdAt": "2024-01-15T10:00:00.000Z",
        "updatedAt": "2024-01-17T14:30:00.000Z"
      },
      "htmlCanvas": {
        "_id": "canvas_2",
        "title": "Landing Page",
        "content": "<!DOCTYPE html>...",
        "description": "",
        "canvasType": "html",
        "currentVersion": 1,
        "versions": [...],
        "createdAt": "2024-01-15T11:00:00.000Z",
        "updatedAt": "2024-01-16T09:00:00.000Z"
      }
    }
  }
}
```

---

### Update Canvas

```http
POST /api/v1/internal/agent-chat-session.canvas-update
```

Update or create a canvas artifact for an AI chat session.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entityId` | string | Yes | Entity ID |
| `itemId` | string | No | Item ID (for item-specific sessions) |
| `roomId` | string | Yes | Room ID |
| `canvasType` | string | Yes | Canvas type: `markdown` or `html` |
| `canvasContent` | string | Yes | Canvas content |
| `title` | string | No | Canvas title (default: "Canvas" or "Landing Page") |

**Request:**

```json
{
  "entityId": "ITEM_123",
  "roomId": "ROOM_456",
  "canvasType": "markdown",
  "canvasContent": "# Updated Content\n\n## Changes\n...",
  "title": "Updated Plan"
}
```

**Response:**

```json
{
  "success": true
}
```

**Behavior:**

1. **New Canvas**: If no canvas exists, creates one with version 1
2. **Update Canvas**: If canvas exists, creates new version with updated content
3. **Version Tracking**: All versions are preserved in `versions` array
4. **WebSocket Event**: Emits `canvas-updated` event for real-time updates

---

## Canvas Types

### Markdown Canvas

```typescript
interface IMarkdownCanvas {
  _id: string;
  title: string;
  content: string;              // Markdown content
  description: string;
  canvasType: 'markdown';
  currentVersion: number;
  versions: IMarkdownCanvasVersion[];
  createdAt: Date;
  updatedAt: Date;
  createdBy: IUserReference;
  updatedBy: IUserReference;
}
```

**Use Cases:**
- Project documentation
- Meeting notes
- Requirement specs
- Knowledge base articles

### HTML Canvas

```typescript
interface IHtmlCanvas {
  _id: string;
  title: string;
  content: string;              // HTML content
  description: string;
  canvasType: 'html';
  currentVersion: number;
  versions: IHtmlCanvasVersion[];
  createdAt: Date;
  updatedAt: Date;
  createdBy: IUserReference;
  updatedBy: IUserReference;
}
```

**Use Cases:**
- Landing pages
- Web prototypes
- UI mockups
- HTML reports

---

## Versioning

Each canvas maintains a version history:

```typescript
interface ICanvasVersion {
  version: number;
  content: string;
  createdAt: Date;
  createdBy: IUserReference;
}
```

**Version Creation:**

- First update → Version 1
- Subsequent updates → Version incremented
- All versions preserved in `versions` array

---

## WebSocket Events

### canvas-updated

Emitted when a canvas is updated:

```typescript
{
  entityType: string;
  entityId: string;
  itemId?: string;
  sessionId: string;
  canvasType: 'markdown' | 'html';
  canvas: IMarkdownCanvas | IHtmlCanvas;
}
```

**Example Client:**

```javascript
RocketChat.streams.subscribe('canvas-updated', roomId, (data) => {
  console.log('Canvas updated:', data.canvas.title);
  console.log('Version:', data.canvas.currentVersion);
});
```

---

## Entity Types

Entity types for context-aware session resume:

| Type | Description | Use Case |
|------|-------------|----------|
| `item` | Individual item/task | AI chat about a specific task or work item |
| `list` | List/board | AI chat about overall list strategy or workflow |
| `file` | File/folder context | AI chat with file browser context |
| `document` | Document | AI chat about document content |
| `room` | Room/conversation | Room-wide AI chat |

When a user opens AI chat with an entity context, the system automatically resolves that context into session filters. For example:
- File browser → uses folder ID as `entityId`
- Items page → uses item ID as `entityId` and `itemId`
- Documents → uses document ID as `entityId`

---

## Error Responses

| Error Code | Description |
|------------|-------------|
| `error-invalid-params` | Missing required parameters |
| `error-invalid-params` | Invalid canvas type (must be "markdown" or "html") |
| `error-invalid-session` | Session not found |
| `Failed to get artifacts` | Failed to retrieve artifacts |

**Error Response Example:**

```json
{
  "success": false,
  "error": "Agent chat session not found"
}
```

---

## Session Object Structure

```typescript
interface IAIChatSession {
  _id: string;
  roomId: string;

  // Context-aware session resume fields
  entityType?: 'item' | 'list' | 'file' | 'document' | 'room';
  entityId?: string;
  entityName?: string;
  itemId?: string;
  attachedContexts?: Array<{
    type: string;
    name: string;
    id?: string;
  }>;

  // Session metadata
  sessionTitle?: string;
  messageCount: number;
  lastMessageAt?: Date;

  // Participants
  participants: Array<{
    userId: string;
    username: string;
    name?: string;
    firstMessageAt: Date;
    lastMessageAt: Date;
    messageCount: number;
  }>;

  // Flowise integration
  flowChatId: string;
  flowSessionId?: string;

  // Canvas artifacts
  markdownCanvas?: IMarkdownCanvas;
  htmlCanvas?: IHtmlCanvas;

  // Timestamps
  createdAt: Date;
  createdBy?: {
    _id: string;
    username: string;
    name?: string;
  };
}
```

---

## Canvas Object Structure

```typescript
interface IUserReference {
  _id: string;
  username: string;
  name?: string;
}

interface IMarkdownCanvas {
  _id: string;
  title: string;
  content: string;
  description: string;
  canvasType: 'markdown';
  versions: Array<{
    version: number;
    content: string;
    createdAt: Date;
    createdBy: IUserReference;
  }>;
  currentVersion: number;
  createdAt: Date;
  createdBy: IUserReference;
  updatedAt: Date;
  updatedBy: IUserReference;
}

interface IHtmlCanvas {
  _id: string;
  title: string;
  content: string;
  description: string;
  canvasType: 'html';
  versions: Array<{
    version: number;
    content: string;
    createdAt: Date;
    createdBy: IUserReference;
  }>;
  currentVersion: number;
  createdAt: Date;
  createdBy: IUserReference;
  updatedAt: Date;
  updatedBy: IUserReference;
}

type CanvasType = 'markdown' | 'html';
```

---

## Related Resources

- [Agent Chat API](./agent-chat.md) - Send bot messages
- [Documents API](./documents.md) - Document versioning
- [Items API](./items.md) - Item management
