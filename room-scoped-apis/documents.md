# Room-Scoped Documents API

## Overview

Manage documents within a specific room. Documents support versioning, templates, and can be organized by template keys.

**Base Path:** `/api/v1/internal/rooms/:roomId/documents`

## Authentication

All endpoints require:
- `x-api-key`: Internal API key
- `Authorization: Bearer <JWT_TOKEN>`: Room-specific JWT token

---

## Endpoints

### Get All Documents

```http
GET /api/v1/internal/rooms/:roomId/documents
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `offset` | number | No | Pagination offset (default: 0) |
| `count` | number | No | Items per page (default: 50, max: 100) |

**Response:**

```json
{
  "success": true,
  "data": {
    "documents": [
      {
        "_id": "DOC_ID",
        "title": "Project Requirements",
        "content": "## Requirements\n\n...",
        "description": "Initial project requirements",
        "roomId": "ROOM_ID",
        "templateKey": "requirements",
        "templateDocumentKey": "req-v1",
        "createdAt": "2024-01-01T00:00:00Z",
        "updatedAt": "2024-01-02T00:00:00Z"
      }
    ],
    "count": 1,
    "offset": 0,
    "total": 1
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/documents?offset=0&count=50" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Create Document

```http
POST /api/v1/internal/rooms/:roomId/documents
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `title` | string | Yes | Document title |
| `content` | string | No | Document content (Markdown supported) |
| `description` | string | No | Document description |
| `templateKey` | string | No | Template category identifier |
| `templateDocumentKey` | string | No | Template version identifier |

**Response:**

```json
{
  "success": true,
  "data": {
    "document": {
      "_id": "NEW_DOC_ID",
      "title": "Meeting Notes",
      "content": "# Meeting Notes\n\n...",
      "roomId": "ROOM_ID",
      "templateKey": "meeting-notes",
      "createdAt": "2024-01-01T00:00:00Z"
    }
  }
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/documents" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Meeting Notes",
    "content": "# Daily Standup\n\n## Attendees\n- John\n- Jane\n\n## Updates\n...",
    "description": "Daily standup meeting notes",
    "templateKey": "meeting-notes",
    "templateDocumentKey": "daily-standup"
  }'
```

---

### Get Document by ID

```http
GET /api/v1/internal/rooms/:roomId/documents/:documentId
```

**Response:**

```json
{
  "success": true,
  "data": {
    "document": {
      "_id": "DOC_ID",
      "title": "Project Requirements",
      "content": "## Requirements\n\n...",
      "description": "Initial project requirements",
      "roomId": "ROOM_ID",
      "templateKey": "requirements",
      "templateDocumentKey": "req-v1"
    }
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/documents/DOC_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Update Document

```http
PUT /api/v1/internal/rooms/:roomId/documents/:documentId
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `title` | string | No | New document title |
| `content` | string | No | New document content |
| `description` | string | No | New description |

**Response:**

```json
{
  "success": true,
  "data": {
    "document": {
      "_id": "DOC_ID",
      "title": "Updated Title",
      "content": "Updated content...",
      "updatedAt": "2024-01-02T12:00:00Z"
    }
  }
}
```

**Example:**

```bash
curl -X PUT "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/documents/DOC_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Updated Title",
    "content": "Updated content..."
  }'
```

---

### Delete Document

```http
DELETE /api/v1/internal/rooms/:roomId/documents/:documentId
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Document deleted successfully",
    "deleted": true
  }
}
```

**Example:**

```bash
curl -X DELETE "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/documents/DOC_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Update Document Content (Versioned)

```http
POST /api/v1/internal/rooms/:roomId/documents/:documentId/update
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `title` | string | No | New title |
| `content` | string | No | New content (creates new version) |
| `description` | string | No | New description |

**Description:** Updates the document and creates a new version in the version history.

**Response:**

```json
{
  "success": true,
  "data": {
    "document": {
      "_id": "DOC_ID",
      "title": "Updated Title",
      "content": "New version content...",
      "versions": [
        {
          "version": 1,
          "content": "Original content...",
          "createdAt": "2024-01-01T00:00:00Z"
        },
        {
          "version": 2,
          "content": "New version content...",
          "createdAt": "2024-01-02T12:00:00Z"
        }
      ],
      "currentVersion": 2
    }
  }
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/documents/DOC_ID/update" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "New version with updates..."
  }'
```

---

## Query by Template

### Get Documents by Template Key

```http
GET /api/v1/internal/rooms/:roomId/documents/byTemplateKey
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `templateKey` | string | Yes | Template key to filter by |

**Response:**

```json
{
  "success": true,
  "data": {
    "documents": [
      {
        "_id": "DOC_ID",
        "templateKey": "meeting-notes",
        "title": "Daily Standup - Jan 1",
        ...
      }
    ],
    "count": 1
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/documents/byTemplateKey?templateKey=meeting-notes" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Get Document by Template Document Key

```http
GET /api/v1/internal/rooms/:roomId/documents/byTemplateDocumentKey
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `templateDocumentKey` | string | Yes | Template document key |

**Response:**

```json
{
  "success": true,
  "data": {
    "document": {
      "_id": "DOC_ID",
      "templateDocumentKey": "daily-standup-v1",
      "title": "Daily Standup Template",
      ...
    }
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/documents/byTemplateDocumentKey?templateDocumentKey=daily-standup-v1" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

## Batch Operations

### Batch Create Documents

```http
POST /api/v1/internal/rooms/:roomId/documents/batch-create
```

**Body Parameters:**

```typescript
{
  documents: Array<{
    title: string,
    content: string,
    description?: string,
    templateKey?: string,
    templateDocumentKey?: string
  }>
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch document creation completed",
    "created": [...],
    "errors": [
      {
        "title": "Invalid Doc",
        "error": "Title and content are required"
      }
    ],
    "totalCreated": 5,
    "totalErrors": 1,
    "processingTime": 2341
  }
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/documents/batch-create" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "documents": [
      {
        "title": "Meeting Notes - Day 1",
        "content": "# Day 1\n\n...",
        "templateKey": "meeting-notes"
      },
      {
        "title": "Meeting Notes - Day 2",
        "content": "# Day 2\n\n...",
        "templateKey": "meeting-notes"
      }
    ]
  }'
```

---

## Error Codes

| Error Code | Description |
|------------|-------------|
| `error-document-not-found` | Document not found |
| `error-forbidden` | Document doesn't belong to this room |
| `error-document-update-failed` | Failed to update document |
| `error-document-deletion-failed` | Failed to delete document |
| `error-invalid-params` | Missing or invalid parameters |

---

## Template System

Documents support a two-level template system:

### Template Key
- Categorizes documents by type
- Examples: `meeting-notes`, `requirements`, `specifications`
- Used for filtering and organization

### Template Document Key
- Identifies a specific template version
- Examples: `daily-standup-v1`, `requirements-v2`
- Used for finding specific template instances

**Example Usage:**

```javascript
// Create a document from a template
POST /documents
{
  "title": "Sprint 1 Requirements",
  "content": "## Sprint 1 Requirements\n\n...",
  "templateKey": "requirements",
  "templateDocumentKey": "sprint-requirements-v1"
}

// Find all requirement documents
GET /documents/byTemplateKey?templateKey=requirements

// Find specific template instance
GET /documents/byTemplateDocumentKey?templateDocumentKey=sprint-requirements-v1
```

---

## Content Format

Documents support **Markdown** content:

```markdown
# Heading 1
## Heading 2
### Heading 3

**Bold text**
*Italic text*

- List item 1
- List item 2

1. Numbered item
2. Another item

`inline code`

```
code block
```

[Link text](https://example.com)
```
