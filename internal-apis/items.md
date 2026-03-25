# Items API

Items are individual tasks, cards, or entities within a List. They can be moved between Stages and have custom field values.

## Base URL

```
/api/v1/internal/items
```

## Endpoints

### Get Items by List ID

```http
GET /api/v1/internal/items.byListId?listId=LIST_ID&offset=0&count=50
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listId` | string | Yes | List ID |
| `offset` | number | No | Pagination offset (default: 0) |
| `count` | number | No | Items per page (default: 50) |

**Response:**

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "_id": "item_1",
        "name": "Implement authentication",
        "key": "implement-auth",
        "description": "Add OAuth2 support",
        "listId": "LIST_ID",
        "stageId": "stage_1",
        "order": 0,
        "customFields": [
          {
            "fieldId": "field_priority",
            "value": "High"
          }
        ],
        "createdAt": "2024-01-15T10:00:00.000Z"
      }
    ],
    "count": 1,
    "offset": 0,
    "total": 25
  }
}
```

---

### Get Items by Stage ID

```http
GET /api/v1/internal/items.byStageId?stageId=STAGE_ID&limit=100
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `stageId` | string | Yes | Stage ID |
| `limit` | number | No | Max items to return (default: 100, max: 200) |

**Response:**

```json
{
  "success": true,
  "data": {
    "items": [...],
    "count": 10,
    "stageId": "stage_1",
    "stageKey": "todo"
  }
}
```

---

### Get Item Info

```http
GET /api/v1/internal/items.info?itemId=ITEM_ID
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemId` | string | Yes | Item ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "item": {
      "_id": "item_1",
      "name": "Parent Task",
      "key": "parent-task",
      "listId": "LIST_ID",
      "stageId": "stage_1",
      "order": 0,
      "customFields": [...],
      "listKey": "backlog",
      "stageKey": "todo",
      "children": [
        {
          "_id": "item_2",
          "name": "Subtask 1",
          "parentId": "item_1"
        }
      ]
    }
  }
}
```

---

### Create Item

```http
POST /api/v1/internal/items.create
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Item name |
| `listId` | string | Yes | List ID |
| `stageId` | string | Yes | Stage ID |
| `description` | string | No | Item description |
| `parentId` | string | No | Parent item ID (for subtasks) |
| `customFields` | array | No | Custom field values |
| `order` | number | No | Position in stage (default: append) |

**Request:**

```json
{
  "name": "Implement API endpoint",
  "listId": "LIST_ID",
  "stageId": "STAGE_ID",
  "description": "Create POST /api/v1/resource endpoint",
  "customFields": [
    {
      "fieldId": "field_priority",
      "value": "High"
    },
    {
      "fieldId": "field_assignee",
      "value": "USER_ID"
    },
    {
      "fieldId": "field_story_points",
      "value": 5
    }
  ],
  "order": 0
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "item": {
      "_id": "item_1",
      "name": "Implement API endpoint",
      "key": "implement-api-endpoint",
      "listId": "LIST_ID",
      "stageId": "STAGE_ID",
      "order": 0,
      "customFields": [...],
      "createdAt": "2024-01-15T10:00:00.000Z"
    }
  }
}
```

---

### Update Item

```http
PUT /api/v1/internal/items.update
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemId` | string | Yes | Item ID |
| `name` | string | No | New item name |
| `description` | string | No | New description |
| `stageId` | string | No | New stage ID |
| `customFields` | array | No | Complete custom fields array (replaces all) |

**Request:**

```json
{
  "itemId": "item_1",
  "name": "Updated name",
  "description": "Updated description",
  "stageId": "stage_2",
  "customFields": [
    {
      "fieldId": "field_priority",
      "value": "Medium"
    }
  ]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Item updated successfully",
    "itemId": "item_1"
  }
}
```

---

### Move Item

```http
POST /api/v1/internal/items.move
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemId` | string | Yes | Item ID |
| `stageId` | string | Yes | Target stage ID |
| `order` | number | No | New position in target stage |

**Request:**

```json
{
  "itemId": "item_1",
  "stageId": "stage_2",
  "order": 5
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Item moved successfully",
    "itemId": "item_1",
    "newStageId": "stage_2"
  }
}
```

---

### Delete Item

```http
DELETE /api/v1/internal/items.delete
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemId` | string | Yes | Item ID |
| `deleteChildren` | boolean | No | Also delete child items (default: false) |

**Request:**

```json
{
  "itemId": "item_1",
  "deleteChildren": true
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Item deleted successfully"
  }
}
```

**Note:** If item has children and `deleteChildren` is false, request will fail with error `error-item-has-children`.

---

## Batch Operations

### Batch Create Items

```http
POST /api/v1/internal/items.batch-create
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listId` | string | Yes | Target list ID |
| `items` | array | Yes | Array of item objects |

**Request:**

```json
{
  "listId": "LIST_ID",
  "items": [
    {
      "name": "Task 1",
      "stageId": "stage_1",
      "customFields": [{"fieldId": "field_priority", "value": "High"}]
    },
    {
      "name": "Task 2",
      "stageId": "stage_1"
    }
  ]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch creation completed",
    "created": [
      {
        "_id": "item_1",
        "name": "Task 1"
      },
      {
        "_id": "item_2",
        "name": "Task 2"
      }
    ],
    "errors": [],
    "totalCreated": 2,
    "totalFailed": 0,
    "processingTime": 234
  }
}
```

---

### Batch Move Items

```http
POST /api/v1/internal/items.batch-move
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemIds` | array | Yes | Array of item IDs |
| `stageId` | string | Yes | Target stage ID |

**Request:**

```json
{
  "itemIds": ["item_1", "item_2", "item_3"],
  "stageId": "stage_2"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch move completed",
    "movedItems": ["item_1", "item_2"],
    "failedItems": [
      {
        "item": {"itemId": "item_3"},
        "error": "Item does not belong to the same list"
      }
    ],
    "totalMoved": 2,
    "totalFailed": 1,
    "processingTime": 123
  }
}
```

---

### Batch Update Items

```http
POST /api/v1/internal/items.batch-update
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `updates` | array | Yes | Array of update objects |

**Request:**

```json
{
  "updates": [
    {
      "itemId": "item_1",
      "name": "Updated Task 1",
      "stageId": "stage_2",
      "customFields": [
        {"fieldId": "field_priority", "value": "Low"}
      ]
    },
    {
      "itemId": "item_2",
      "order": 5
    }
  ]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch update completed",
    "updated": [
      {
        "_id": "item_1",
        "name": "Updated Task 1"
      }
    ],
    "failed": [
      {
        "item": {"itemId": "item_2"},
        "error": "Stage not found"
      }
    ],
    "totalUpdated": 1,
    "totalFailed": 1,
    "processingTime": 456
  }
}
```

**Note:** For more than 200 updates, processing is automatically chunked.

---

## Custom Fields Operations

### Get All Custom Fields

```http
GET /api/v1/internal/items/:itemId/customFields
```

**Response:**

```json
{
  "success": true,
  "data": {
    "itemId": "item_1",
    "customFields": [
      {
        "fieldId": "field_priority",
        "value": "High",
        "definition": {
          "_id": "field_priority",
          "name": "Priority",
          "type": "SELECT",
          "options": ["High", "Medium", "Low"]
        }
      }
    ]
  }
}
```

---

### Update Single Custom Field

```http
PUT /api/v1/internal/items/:itemId/customFields/:fieldId
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `value` | any | Yes | New field value |

**Request:**

```json
{
  "value": "Medium"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Custom field updated successfully",
    "itemId": "item_1",
    "fieldId": "field_priority",
    "value": "Medium",
    "item": { /* Full updated item */ }
  }
}
```

---

### Delete Custom Field

```http
DELETE /api/v1/internal/items/:itemId/customFields/:fieldId
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Custom field removed successfully",
    "itemId": "item_1",
    "fieldId": "field_priority"
  }
}
```

---

### Update Multiple Custom Fields (Partial)

```http
PUT /api/v1/internal/items/:itemId/customFields
```

**OR**

```http
POST /api/v1/internal/items/:itemId/customFields
```

> **Note:** Both PUT and POST methods are supported and behave identically.

**URL Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemId` | string | Yes | Item ID |


**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fields` | array | Yes | Array of field updates |

**Request:**

```json
{
  "fields": [
    {"fieldId": "field_priority", "value": "High"},
    {"fieldId": "field_assignee", "value": "USER_ID"},
    {"fieldId": "field_due_date", "value": "2024-01-30"}
  ]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Custom fields updated successfully",
    "itemId": "item_1",
    "updatedFields": 3,
    "item": { /* Full updated item */ }
  }
}
```

**Behavior:**

This is a **partial update (merge)** operation:

| Scenario | Behavior |
|----------|----------|
| **Field already exists** | ✅ Updated with new value (overwrites existing) |
| **Field doesn't exist** | ➕ Added to the item |
| **Field not in request** | 🔒 Unchanged (remains as-is) |

**Example:**

```json
// Current item has: [{fieldId: "priority", value: "Low"}, {fieldId: "status", value: "Open"}]

// Request:
{
  "fields": [
    {"fieldId": "priority", "value": "High"}  // Update existing
    // No "status" field - remains "Open"
  ]
}

// Result: [{fieldId: "priority", value: "High"}, {fieldId: "status", value: "Open"}]
```

---

## Key-Based Operations

### Get Item by Key

```http
GET /api/v1/internal/items.byKey?itemKey=TASK_KEY&roomId=ROOM_ID&listTemplateKey=backlog
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemKey` | string | Yes | Item unique key |
| `roomId` | string | Yes | Room ID |
| `listTemplateKey` | string | Yes | List template key |

---

### Update Item by Key

```http
PUT /api/v1/internal/items.updateByKey
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemKey` | string | Yes | Item key |
| `roomId` | string | Yes | Room ID |
| `listTemplateKey` | string | Yes | List template key |
| `name` | string | No | New name |
| `description` | string | No | New description |
| `customFields` | array | No | Custom fields |

---

### Move Item by Key

```http
POST /api/v1/internal/items.moveByKey
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemKey` | string | Yes | Item key |
| `stageKey` | string | Yes | Target stage key |
| `roomId` | string | Yes | Room ID |
| `listTemplateKey` | string | Yes | List template key |
| `order` | number | No | New position |

---

### Delete Item by Key

```http
DELETE /api/v1/internal/items.deleteByKey
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemKey` | string | Yes | Item key |
| `roomId` | string | Yes | Room ID |
| `listTemplateKey` | string | Yes | List template key |
| `deleteChildren` | boolean | No | Delete child items |

---

### Change Stage by Keys

```http
POST /api/v1/internal/items.changeStageByKeys
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemId` | string | Yes | Item ID |
| `stageKey` | string | Yes | Target stage key |
| `roomId` | string | Yes | Room ID |
| `listTemplateKey` | string | Yes | List template key |
| `order` | number | No | New position |

---

## Error Responses

| Error Code | Description |
|------------|-------------|
| `error-invalid-params` | Missing or invalid parameters |
| `error-item-not-found` | Item not found |
| `error-list-not-found` | List not found |
| `error-stage-not-found` | Stage not found |
| `error-invalid-stage` | Stage doesn't belong to list |
| `error-invalid-parent` | Parent item invalid |
| `error-invalid-field` | Custom field not found in list |
| `error-item-has-children` | Item has children (requires deleteChildren=true) |

## Related Resources

- [Lists API](./lists.md) - Manage lists containing items
- [Stages API](./stages.md) - Manage stages within lists
