# Room-Scoped Items API

## Overview

Manage items within lists in a specific room. Items represent tasks, tickets, or cards that can be moved between stages and have custom field values.

**Base Path:** `/api/v1/internal/rooms/:roomId/items`

## Authentication

All endpoints require:
- `x-api-key`: Internal API key
- `Authorization: Bearer <JWT_TOKEN>`: Room-specific JWT token

---

## Endpoints

### Get Items

```http
GET /api/v1/internal/rooms/:roomId/items
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listId` | string | Conditional* | List ID (required if stageId not provided) |
| `stageId` | string | Conditional* | Stage ID (required if listId not provided) |
| `offset` | number | No | Pagination offset (default: 0) |
| `count` | number | No | Items per page (default: 50) |

*Either `listId` or `stageId` must be provided.

**Response:**

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "_id": "ITEM_ID",
        "name": "Implement user authentication",
        "description": "Add OAuth2 support",
        "listId": "LIST_ID",
        "stageId": "STAGE_ID",
        "parentId": null,
        "customFields": [
          {
            "fieldId": "FIELD_ID",
            "value": "High"
          }
        ],
        "order": 0,
        "createdAt": "2024-01-01T00:00:00Z"
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
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/items?listId=LIST_ID&offset=0&count=50" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Create Item

```http
POST /api/v1/internal/rooms/:roomId/items
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Item name |
| `description` | string | No | Item description |
| `listId` | string | Yes | Parent list ID |
| `stageId` | string | Yes | Initial stage ID |
| `parentId` | string | No | Parent item ID (for nested items) |
| `customFields` | array | No | Array of custom field values |
| `order` | number | No | Display order |

**Custom Field Value Object:**

```typescript
{
  fieldId: string,    // Field definition ID
  value: any          // Field value (type depends on field definition)
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "item": {
      "_id": "NEW_ITEM_ID",
      "name": "Implement user authentication",
      "listId": "LIST_ID",
      "stageId": "STAGE_ID",
      "customFields": [...]
    }
  }
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/items" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Implement user authentication",
    "description": "Add OAuth2 support",
    "listId": "LIST_ID",
    "stageId": "STAGE_ID",
    "customFields": [
      {
        "fieldId": "FIELD_ID",
        "value": "High"
      }
    ]
  }'
```

---

### Get Item by ID

```http
GET /api/v1/internal/rooms/:roomId/items/:itemId
```

**Response:**

```json
{
  "success": true,
  "data": {
    "item": {
      "_id": "ITEM_ID",
      "name": "Implement user authentication",
      "description": "Add OAuth2 support",
      "listId": "LIST_ID",
      "stageId": "STAGE_ID",
      "customFields": [...],
      "order": 0,
      "children": [
        {
          "_id": "CHILD_ITEM_ID",
          "name": "Subtask 1",
          ...
        }
      ],
      "listKey": "sprint-backlog",
      "stageKey": "in-progress"
    }
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/items/ITEM_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Update Item

```http
PUT /api/v1/internal/rooms/:roomId/items/:itemId
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | No | New item name |
| `description` | string | No | New description |
| `customFields` | array | No | New custom field values |
| `stageId` | string | No | Move to different stage |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Item updated successfully",
    "item": { ... }
  }
}
```

**Example:**

```bash
curl -X PUT "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/items/ITEM_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated item name",
    "stageId": "NEW_STAGE_ID"
  }'
```

---

### Delete Item

```http
DELETE /api/v1/internal/rooms/:roomId/items/:itemId
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `deleteChildren` | boolean | No | Also delete child items (default: false) |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Item deleted successfully",
    "deleted": true
  }
}
```

**Error (if has children and deleteChildren=false):**

```json
{
  "success": false,
  "error": "error-item-has-children",
  "message": "Item has 3 children. Set deleteChildren to true to delete them."
}
```

**Example:**

```bash
curl -X DELETE "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/items/ITEM_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "deleteChildren": true
  }'
```

---

### Move Item

```http
POST /api/v1/internal/rooms/:roomId/items/:itemId/move
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `stageId` | string | Yes | Target stage ID |
| `order` | number | No | New position order |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Item moved successfully",
    "itemId": "ITEM_ID",
    "newStageId": "NEW_STAGE_ID"
  }
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/items/ITEM_ID/move" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "stageId": "TARGET_STAGE_ID",
    "order": 5
  }'
```

---

## Custom Fields Management

### Get Custom Fields

```http
GET /api/v1/internal/rooms/:roomId/items/:itemId/customFields
```

**Response:**

```json
{
  "success": true,
  "data": {
    "itemId": "ITEM_ID",
    "customFields": [
      {
        "fieldId": "FIELD_ID",
        "value": "High",
        "definition": {
          "_id": "FIELD_ID",
          "name": "Priority",
          "type": "SELECT",
          "options": [...]
        }
      }
    ]
  }
}
```

---

### Update Custom Fields (Batch)

```http
PUT /api/v1/internal/rooms/:roomId/items/:itemId/customFields
```

**Body Parameters:**

```typescript
{
  fields: Array<{
    fieldId: string,
    value: any
  }>
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Custom fields updated successfully",
    "itemId": "ITEM_ID",
    "updatedFields": 2,
    "item": { ... }
  }
}
```

**Example:**

```bash
curl -X PUT "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/items/ITEM_ID/customFields" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "fields": [
      {
        "fieldId": "FIELD_ID_1",
        "value": "High"
      },
      {
        "fieldId": "FIELD_ID_2",
        "value": "Backend"
      }
    ]
  }'
```

---

### Update Single Custom Field

```http
PUT /api/v1/internal/rooms/:roomId/items/:itemId/customFields/:fieldId
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `value` | any | Yes | New field value |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Custom field updated successfully",
    "itemId": "ITEM_ID",
    "fieldId": "FIELD_ID",
    "value": "High",
    "item": { ... }
  }
}
```

---

### Delete Custom Field Value

```http
DELETE /api/v1/internal/rooms/:roomId/items/:itemId/customFields/:fieldId
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Custom field removed successfully",
    "itemId": "ITEM_ID",
    "fieldId": "FIELD_ID"
  }
}
```

---

## Batch Operations

### Batch Create Items

```http
POST /api/v1/internal/rooms/:roomId/items/batch-create
```

**Body Parameters:**

```typescript
{
  listId: string,
  items: Array<{
    name: string,
    description?: string,
    stageId: string,
    parentId?: string,
    customFields?: Array<{ fieldId: string, value: any }>,
    order?: number
  }>
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch creation completed",
    "created": [...],
    "errors": [
      {
        "item": { ... },
        "error": "Invalid stage ID"
      }
    ],
    "totalCreated": 8,
    "totalFailed": 1,
    "processingTime": 2341
  }
}
```

---

### Batch Move Items

```http
POST /api/v1/internal/rooms/:roomId/items/batch-move
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemIds` | array | Yes | Array of item IDs to move |
| `stageId` | string | Yes | Target stage ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch move completed",
    "movedItems": ["ID1", "ID2", "ID3"],
    "failedItems": [
      {
        "item": { "itemId": "ID4" },
        "error": "Item does not belong to this room"
      }
    ],
    "totalMoved": 3,
    "totalFailed": 1,
    "processingTime": 1234
  }
}
```

---

### Batch Update Items

```http
POST /api/v1/internal/rooms/:roomId/items/batch-update
```

**Body Parameters:**

```typescript
{
  updates: Array<{
    itemId: string,
    name?: string,
    description?: string,
    stageId?: string,
    customFields?: Array<{ fieldId: string, value: any }>,
    order?: number
  }>
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch update completed",
    "updated": [...],
    "errors": [...],
    "totalUpdated": 10,
    "totalFailed": 0,
    "processingTime": 3456
  }
}
```

---

### Batch Delete Items

```http
POST /api/v1/internal/rooms/:roomId/items/batch-delete
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemIds` | array | Yes | Array of item IDs to delete |
| `deleteChildren` | boolean | No | Also delete child items (default: false) |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch deletion completed",
    "deleted": ["ID1", "ID2"],
    "errors": [
      {
        "item": { "itemId": "ID3" },
        "error": "Item has 2 children. Set deleteChildren to true."
      }
    ],
    "totalDeleted": 2,
    "totalFailed": 1,
    "processingTime": 1234
  }
}
```

---

## Error Codes

| Error Code | Description |
|------------|-------------|
| `error-item-not-found` | Item not found |
| `error-list-not-found` | List not found in this room |
| `error-stage-not-found` | Stage not found |
| `error-invalid-stage` | Stage doesn't belong to the item's list |
| `error-invalid-parent` | Parent item not found or doesn't belong to list |
| `error-invalid-field` | Field not found in list |
| `error-item-has-children` | Item has child items |
| `error-forbidden` | Item doesn't belong to this room |

---

## Item Hierarchy

Items support parent-child relationships for nested tasks:

```
Parent Item
├── Child Item 1
│   └── Grandchild Item
└── Child Item 2
```

**Rules:**
- Child items must belong to the same list as parent
- Deleting a parent requires `deleteChildren: true`
- Moving a parent doesn't automatically move children
