# Room-Scoped Lists API

## Overview

Manage Kanban-style lists within a specific room. Lists contain custom field definitions and stages for organizing items.

**Base Path:** `/api/v1/internal/rooms/:roomId/lists`

## Authentication

All endpoints require:
- `x-api-key`: Internal API key
- `Authorization: Bearer <JWT_TOKEN>`: Room-specific JWT token

---

## Endpoints

### Get All Lists

```http
GET /api/v1/internal/rooms/:roomId/lists
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `text` | string | No | Filter lists by name |
| `offset` | number | No | Pagination offset (default: 0) |
| `count` | number | No | Items per page (default: 50, max: 100) |

**Response:**

```json
{
  "success": true,
  "data": {
    "roomId": "ROOM_ID",
    "lists": [
      {
        "_id": "LIST_ID",
        "name": "Sprint Backlog",
        "key": "sprint-backlog",
        "roomId": "ROOM_ID",
        "templateKey": "sprint-backlog",
        "templateListKey": "sprint-backlog-v1",
        "description": "Current sprint items",
        "fieldDefinitions": [
          {
            "_id": "FIELD_ID",
            "name": "Priority",
            "type": "SELECT",
            "options": [
              { "_id": "OPT1", "value": "High", "color": "#ef4444" },
              { "_id": "OPT2", "value": "Medium", "color": "#f59e0b" }
            ],
            "order": 0
          }
        ],
        "stageCount": 3,
        "itemCount": 15
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
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/lists?offset=0&count=50" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Create List

```http
POST /api/v1/internal/rooms/:roomId/lists
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | List name |
| `description` | string | No | List description |
| `fieldDefinitions` | array | No | Array of field definition objects |
| `stages` | array | No | Array of stage objects |

**Field Definition Object:**

```typescript
{
  _id?: string,           // Auto-generated if not provided
  name: string,           // Field name
  type: string,           // TEXT, SELECT, MULTI_SELECT, DATE, NUMBER
  options?: Array<{       // For SELECT/MULTI_SELECT only
    _id?: string,
    value: string,
    color?: string,
    order?: number
  }>,
  order?: number,         // Display order
  displayFormat?: string  // Optional display format
}
```

**Stage Object:**

```typescript
{
  name: string,
  color?: string,         // Hex color (default: #6b7280)
  order?: number          // Display order
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "list": {
      "_id": "NEW_LIST_ID",
      "name": "Sprint Backlog",
      "roomId": "ROOM_ID",
      "key": "sprint-backlog"
    }
  }
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/lists" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sprint Backlog",
    "description": "Current sprint items",
    "fieldDefinitions": [
      {
        "name": "Priority",
        "type": "SELECT",
        "options": [
          { "value": "High", "color": "#ef4444" },
          { "value": "Medium", "color": "#f59e0b" },
          { "value": "Low", "color": "#10b981" }
        ]
      }
    ],
    "stages": [
      { "name": "To Do", "color": "#6b7280" },
      { "name": "In Progress", "color": "#3b82f6" },
      { "name": "Done", "color": "#10b981" }
    ]
  }'
```

---

### Get List by ID

```http
GET /api/v1/internal/rooms/:roomId/lists/:listId
```

**Response:**

```json
{
  "success": true,
  "data": {
    "list": {
      "_id": "LIST_ID",
      "name": "Sprint Backlog",
      "key": "sprint-backlog",
      "roomId": "ROOM_ID",
      "fieldDefinitions": [...]
    },
    "stages": [...],
    "items": [...]
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/lists/LIST_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Update List

```http
PUT /api/v1/internal/rooms/:roomId/lists/:listId
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | No | New list name |
| `description` | string | No | New description |

**Note:** If name is changed and list has no items, the `key` is auto-regenerated.

**Response:**

```json
{
  "success": true,
  "data": {
    "list": {
      "_id": "LIST_ID",
      "name": "Updated Name",
      "key": "updated-name",
      "description": "New description"
    }
  }
}
```

**Example:**

```bash
curl -X PUT "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/lists/LIST_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Sprint Backlog",
    "description": "Updated description"
  }'
```

---

### Delete List

```http
DELETE /api/v1/internal/rooms/:roomId/lists/:listId
```

**Description:** Deletes a list and all its associated stages and items.

**Response:**

```json
{
  "success": true,
  "data": {
    "deleted": true
  }
}
```

**Example:**

```bash
curl -X DELETE "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/lists/LIST_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Get Lists by Template Key

```http
GET /api/v1/internal/rooms/:roomId/lists/byTemplateKey
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
    "lists": [
      {
        "_id": "LIST_ID",
        "templateKey": "sprint-backlog",
        ...
      }
    ]
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/lists/byTemplateKey?templateKey=sprint-backlog" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Get List by Template List Key

```http
GET /api/v1/internal/rooms/:roomId/lists/byTemplateListKey
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `templateListKey` | string | Yes | Template list key to filter by |

**Response:**

```json
{
  "success": true,
  "data": {
    "list": { ... },
    "stages": [ ... ],
    "items": [ ... ]
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/lists/byTemplateListKey?templateListKey=sprint-backlog-v1" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

## Field Definitions Management

### Get All Field Definitions

```http
GET /api/v1/internal/rooms/:roomId/lists/:listId/fields
```

**Response:**

```json
{
  "success": true,
  "data": {
    "fieldDefinitions": [
      {
        "_id": "FIELD_ID",
        "name": "Priority",
        "type": "SELECT",
        "options": [...],
        "order": 0
      }
    ]
  }
}
```

---

### Add Field Definition

```http
POST /api/v1/internal/rooms/:roomId/lists/:listId/fields
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `field` | object | Yes | Field definition object |

**Field Object:**

```typescript
{
  _id?: string,
  name: string,
  type: 'TEXT' | 'SELECT' | 'MULTI_SELECT' | 'DATE' | 'NUMBER',
  options?: Array<{ value: string, color?: string }>,
  order?: number,
  displayFormat?: string
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Field added successfully",
    "fieldId": "NEW_FIELD_ID",
    "field": { ... }
  }
}
```

---

### Update All Field Definitions

```http
PUT /api/v1/internal/rooms/:roomId/lists/:listId/fields
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fieldDefinitions` | array | Yes | Array of field definition objects |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Field definitions updated successfully",
    "fieldDefinitions": [...]
  }
}
```

---

### Get Single Field Definition

```http
GET /api/v1/internal/rooms/:roomId/lists/:listId/fields/:fieldId
```

**Response:**

```json
{
  "success": true,
  "data": {
    "field": { ... }
  }
}
```

---

### Update Single Field Definition

```http
PUT /api/v1/internal/rooms/:roomId/lists/:listId/fields/:fieldId
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `data` | object | Yes | Partial field update |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Field updated successfully",
    "field": { ... }
  }
}
```

---

### Delete Field Definition

```http
DELETE /api/v1/internal/rooms/:roomId/lists/:listId/fields/:fieldId
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Field deleted successfully",
    "fieldId": "FIELD_ID"
  }
}
```

---

### Add Option to Select Field

```http
POST /api/v1/internal/rooms/:roomId/lists/:listId/fields/:fieldId/options
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `value` | string | Yes | Option value |
| `color` | string | No | Hex color (default: #3498db) |
| `order` | number | No | Display order |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Option added successfully",
    "option": {
      "_id": "OPTION_ID",
      "value": "New Option",
      "color": "#3498db",
      "order": 0
    }
  }
}
```

---

### Update Field Option

```http
PUT /api/v1/internal/rooms/:roomId/lists/:listId/fields/:fieldId/options/:optionId
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `value` | string | No | New option value |
| `color` | string | No | New hex color |
| `order` | number | No | New display order |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Option updated successfully"
  }
}
```

---

### Delete Field Option

```http
DELETE /api/v1/internal/rooms/:roomId/lists/:listId/fields/:fieldId/options/:optionId
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Option deleted successfully"
  }
}
```

---

## Batch Operations

### Batch Create Lists

```http
POST /api/v1/internal/rooms/:roomId/lists/batch-create
```

**Body Parameters:**

```typescript
{
  lists: Array<{
    name: string,
    description?: string,
    fieldDefinitions?: FieldDefinition[],
    stages?: Array<{
      name: string,
      color?: string,
      order?: number
    }>,
    templateKey?: string,
    templateListKey?: string
  }>
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch lists creation completed",
    "created": [...],
    "errors": [
      {
        "list": { ... },
        "error": "Error message"
      }
    ],
    "totalCreated": 5,
    "totalFailed": 1,
    "processingTime": 1234
  }
}
```

---

### Batch Delete Lists

```http
POST /api/v1/internal/rooms/:roomId/lists/batch-delete
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listIds` | array | Yes | Array of list IDs to delete |
| `deleteItems` | boolean | No | Also delete items (default: false) |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch lists deletion completed",
    "deleted": ["LIST_ID_1", "LIST_ID_2"],
    "errors": [...],
    "totalDeleted": 2,
    "totalFailed": 0,
    "processingTime": 567
  }
}
```

---

## Error Codes

| Error Code | Description |
|------------|-------------|
| `error-list-not-found` | List not found or doesn't belong to this room |
| `error-invalid-params` | Missing or invalid parameters |
| `error-forbidden` | List doesn't belong to this room |
| `error-field-not-found` | Field definition not found |
| `error-option-not-found` | Field option not found |
| `error-invalid-field-type` | Field doesn't support the operation |

---

## Field Types

| Type | Description | Supports Options |
|------|-------------|------------------|
| `TEXT` | Plain text input | No |
| `SELECT` | Single select dropdown | Yes |
| `MULTI_SELECT` | Multi-select dropdown | Yes |
| `DATE` | Date picker | No |
| `NUMBER` | Numeric input | No |
