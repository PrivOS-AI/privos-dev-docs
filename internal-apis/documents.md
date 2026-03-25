# Documents API

Documents are versioned content entities that support rich text editing and version history tracking.

## Base URL

```
/api/v1/internal/documents
```

## Endpoints

### Get Document by ID

```http
GET /api/v1/internal/documents/:documentId
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `documentId` | string | Yes | Document ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "document": {
      "_id": "doc_1",
      "roomId": "ROOM_ID",
      "title": "Project Requirements",
      "content": "# Requirements\n\n## Overview\n...",
      "description": "Initial requirements doc",
      "currentVersion": 3,
      "createdAt": "2024-01-15T10:00:00.000Z",
      "updatedAt": "2024-01-17T14:30:00.000Z"
    }
  }
}
```

---

### Update Document (Simple)

```http
PUT /api/v1/internal/documents/:documentId
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `documentId` | string | Yes | Document ID |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `title` | string | No | New document title |
| `content` | string | No | New content (creates version) |
| `description` | string | No | New description |

**Request:**

```json
{
  "title": "Updated Requirements",
  "content": "# Updated Requirements\n\n## Changes\n...",
  "description": "Updated after review"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "document": {
      "_id": "doc_1",
      "title": "Updated Requirements",
      "currentVersion": 4,
      "updatedAt": "2024-01-18T09:00:00.000Z"
    }
  }
}
```

---

### Delete Document

```http
DELETE /api/v1/internal/documents/:documentId
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `documentId` | string | Yes | Document ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "deleted": true
  }
}
```

---

### Update Document Content

```http
POST /api/v1/internal/documents.update
```

Update document with automatic version creation.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `documentId` | string | Yes | Document ID |
| `title` | string | No | New title |
| `content` | string | No | New content (creates new version) |
| `description` | string | No | New description |

**Request:**

```json
{
  "documentId": "doc_1",
  "content": "# New Content\n\nUpdated section...",
  "title": "New Title"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "document": {
      "_id": "doc_1",
      "title": "New Title",
      "currentVersion": 5,
      "updatedAt": "2024-01-18T10:00:00.000Z"
    }
  }
}
```

---

### Get Documents by Room ID

```http
GET /api/v1/internal/documents.byRoomId?roomId=ROOM_ID&offset=0&count=50
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `offset` | number | No | Pagination offset |
| `count` | number | No | Items per page |

**Response:**

```json
{
  "success": true,
  "data": {
    "documents": [
      {
        "_id": "doc_1",
        "title": "Requirements",
        "currentVersion": 3,
        "updatedAt": "2024-01-17T14:30:00.000Z"
      }
    ],
    "count": 1,
    "offset": 0,
    "total": 5
  }
}
```

---

### Get Documents by Template Key

```http
GET /api/v1/internal/documents.byTemplateKey?templateKey=requirements&roomId=ROOM_ID
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `templateKey` | string | Yes | Template key |
| `roomId` | string | Yes | Room ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "documents": [
      {
        "_id": "doc_1",
        "templateDocumentKey": "requirements-v1",
        "title": "Requirements",
        "content": "..."
      }
    ]
  }
}
```

---

### Get Document by Template Document Key

```http
GET /api/v1/internal/documents.byTemplateDocumentKey?templateDocumentKey=REQ_TEMPLATE&roomId=ROOM_ID
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `templateDocumentKey` | string | Yes | Template document key |
| `roomId` | string | Yes | Room ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "document": {
      "_id": "doc_1",
      "templateDocumentKey": "REQ_TEMPLATE",
      "title": "Requirements",
      "content": "..."
    }
  }
}
```

---

### Get Version History

```http
GET /api/v1/internal/documents.versions?documentId=doc_1
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `documentId` | string | Yes | Document ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "documentId": "doc_1",
    "currentVersion": 3,
    "versions": [
      {
        "version": 1,
        "createdAt": "2024-01-15T10:00:00.000Z",
        "createdBy": "john.doe"
      },
      {
        "version": 2,
        "createdAt": "2024-01-16T14:00:00.000Z",
        "createdBy": "jane.smith"
      },
      {
        "version": 3,
        "createdAt": "2024-01-17T09:00:00.000Z",
        "createdBy": "ai-assistant"
      }
    ]
  }
}
```

---

### Get Specific Version

```http
GET /api/v1/internal/documents.getVersion?documentId=doc_1&version=2
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `documentId` | string | Yes | Document ID |
| `version` | string | Yes | Version number |

**Response:**

```json
{
  "success": true,
  "data": {
    "documentTitle": "Project Requirements",
    "version": {
      "version": 2,
      "content": "# Requirements\n\n## V2 Content\n...",
      "createdAt": "2024-01-16T14:00:00.000Z",
      "createdBy": "jane.smith"
    }
  }
}
```

---

### Create Document

```http
POST /api/v1/internal/documents.create
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `title` | string | Yes | Document title |
| `content` | string | Yes | Document content |
| `description` | string | No | Document description |

**Request:**

```json
{
  "roomId": "ROOM_ID",
  "title": "Meeting Notes",
  "content": "# Meeting Notes - 2024-01-15\n\n## Attendees\n...",
  "description": "Weekly team sync notes"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "document": {
      "_id": "doc_1",
      "title": "Meeting Notes",
      "content": "...",
      "currentVersion": 1,
      "createdAt": "2024-01-15T10:00:00.000Z"
    }
  }
}
```

---

### Revert to Version

```http
POST /api/v1/internal/documents.revert
```

Revert document to a previous version (creates new version with old content).

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `documentId` | string | Yes | Document ID |
| `version` | string | Yes | Version to revert to |

**Request:**

```json
{
  "documentId": "doc_1",
  "version": "2"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "document": {
      "_id": "doc_1",
      "title": "Project Requirements",
      "currentVersion": 4,
      "content": "...", // Content from version 2
      "updatedAt": "2024-01-18T11:00:00.000Z"
    }
  }
}
```

---

## Batch Operations

### Batch Create Documents

```http
POST /api/v1/internal/documents.batch-create
```

**Rate Limit:** 5 requests per minute

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Target room ID |
| `documents` | array | Yes | Array of document objects |

**Request:**

```json
{
  "roomId": "ROOM_ID",
  "documents": [
    {
      "title": "Doc 1",
      "content": "Content 1...",
      "description": "Description 1"
    },
    {
      "title": "Doc 2",
      "content": "Content 2..."
    }
  ]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch document creation completed",
    "created": [
      {
        "_id": "doc_1",
        "title": "Doc 1"
      },
      {
        "_id": "doc_2",
        "title": "Doc 2"
      }
    ],
    "errors": [
      {
        "title": "Doc 3",
        "error": "Title is required"
      }
    ],
    "totalCreated": 2,
    "totalErrors": 1,
    "processingTime": 1234
  }
}
```

---

## Document Object

```typescript
interface IDocument {
  _id: string;
  roomId: string;
  title: string;
  content: string;
  description?: string;
  currentVersion: number;
  templateKey?: string;
  templateDocumentKey?: string;
  versions: IDocumentVersion[];
  createdAt: Date;
  updatedAt: Date;
  createdBy: IUserReference;
  updatedBy: IUserReference;
}

interface IDocumentVersion {
  version: number;
  content: string;
  createdAt: Date;
  createdBy: IUserReference;
}

interface IUserReference {
  _id: string;
  username: string;
  name?: string;
}
```

## Error Responses

| Error Code | Description |
|------------|-------------|
| `error-invalid-params` | Document ID or required parameters missing |
| `error-document-not-found` | Document not found |
| `error-document-update-failed` | Failed to update document |
| `error-document-deletion-failed` | Failed to delete document |
| `error-document-creation-failed` | Failed to create document |
| `error-version-not-found` | Requested version not found |

## Versioning Behavior

- **Automatic Version Creation**: Updating `content` via `PUT` or `POST` creates a new version
- **Version Numbers**: Start at 1, increment with each content change
- **Version History**: All versions are preserved and accessible
- **Revert**: Creates a new version with content from the target version
- **Metadata Changes**: Updating `title` or `description` alone does NOT create a new version
