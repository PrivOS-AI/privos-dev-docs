# Room-Scoped Lists API

## Overview

Manage Kanban-style lists within a specific room. Lists contain custom field definitions and stages for organizing items.

When the target room belongs to a team, the GET endpoints can also surface cross-team workflow lists shared into that team context. Those shared lists are marked with `isCrossTeamFromOtherRoom: true`.

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
| `text` | string | No | Filter lists by name or description |
| `offset` | number | No | Pagination offset (default: 0) |
| `count` | number | No | Items per page (default: 50, max: 100) |

**Behavior:**

- If `roomId` is a standalone room, this endpoint returns lists that belong directly to that room.
- If `roomId` belongs to a team, this endpoint returns:
  - lists from the team main room and team child rooms
  - cross-team lists shared into that team via stage assignment
- Shared lists are marked with `isCrossTeamFromOtherRoom: true`.
- Local/team-owned lists are marked with `isCrossTeamFromOtherRoom: false`.

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
        "crossTeamWorkflow": true,
        "isCrossTeamFromOtherRoom": false,
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
| `crossTeamWorkflow` | boolean | No | Enable cross-team workflow for the list. Requires `manage-cross-team-workflow` permission when set to `true`. |
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
    "crossTeamWorkflow": true,
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

**Behavior:**

- Returns a local list when the list belongs directly to `roomId`.
- When `roomId` belongs to a team, this endpoint can also return a cross-team list shared into that team.
- Cross-team shared lists are marked with `isCrossTeamFromOtherRoom: true`.
- For cross-team shared lists, the returned `stages` are filtered to the current team context.

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
      "crossTeamWorkflow": true,
      "isCrossTeamFromOtherRoom": false,
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
| `crossTeamWorkflow` | boolean | No | Enable or disable cross-team workflow. Changing this value requires `manage-cross-team-workflow` permission. |

**Note:** If name is changed and list has no items, the `key` is auto-regenerated.

**Important:** This update route still applies only to lists whose `roomId` matches the room in the URL. Cross-team shared lists are readable through team context, but are not updated through the shared room-scoped route.

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
    "description": "Updated description",
    "crossTeamWorkflow": false
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
| `error-list-not-found` | List not found |
| `error-invalid-params` | Missing or invalid parameters |
| `error-forbidden` | List is not visible in this room or team context |
| `error-unauthorized` | Missing permission to enable or modify cross-team workflow |
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
