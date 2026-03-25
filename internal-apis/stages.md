# Stages API

Stages represent workflow states (like "To Do", "In Progress", "Done") within a List. Items are organized into stages.

## Base URL

```
/api/v1/internal/stages
```

## Endpoints

### Get Stages by List ID

```http
GET /api/v1/internal/stages.byListId?listId=LIST_ID
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listId` | string | Yes | List ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "stages": [
      {
        "_id": "stage_1",
        "name": "To Do",
        "listId": "LIST_ID",
        "color": "#6b7280",
        "order": 0,
        "templateStageKey": "todo",
        "itemCount": 15
      },
      {
        "_id": "stage_2",
        "name": "In Progress",
        "listId": "LIST_ID",
        "color": "#3b82f6",
        "order": 1,
        "templateStageKey": "in-progress",
        "itemCount": 5
      }
    ],
    "count": 2
  }
}
```

---

### Get Stage by ID

```http
GET /api/v1/internal/stages/:stageId
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `stageId` | string | Yes | Stage ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "stage": {
      "_id": "stage_1",
      "name": "To Do",
      "listId": "LIST_ID",
      "color": "#6b7280",
      "order": 0
    },
    "itemCount": 15
  }
}
```

---

### Update Stage

```http
PUT /api/v1/internal/stages/:stageId
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `stageId` | string | Yes | Stage ID |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | No | New stage name |
| `color` | string | No | New stage color (hex format) |

**Request:**

```json
{
  "name": "Updated Stage Name",
  "color": "#10b981"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Stage updated successfully",
    "stageId": "stage_1"
  }
}
```

---

### Delete Stage

```http
DELETE /api/v1/internal/stages/:stageId
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `stageId` | string | Yes | Stage ID |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `moveItemsToStageId` | string | No | Target stage for existing items |

**Request (with items to move):**

```json
{
  "moveItemsToStageId": "stage_2"
}
```

**Response (empty stage):**

```json
{
  "success": true,
  "data": {
    "message": "Stage deleted successfully"
  }
}
```

**Response (stage with items, no target):**

```json
{
  "success": false,
  "error": "error-stage-has-items",
  "message": "Stage has 15 items. Provide moveItemsToStageId to move them."
}
```

**Note:** If stage contains items, you MUST provide `moveItemsToStageId`. Items will be moved to the specified stage before deletion.

---

### Create Stage

```http
POST /api/v1/internal/stages.create
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listId` | string | Yes | Parent list ID |
| `name` | string | Yes | Stage name |
| `color` | string | Yes | Stage color (hex format) |
| `order` | number | No | Position in list (default: append) |

**Request:**

```json
{
  "listId": "LIST_ID",
  "name": "Review",
  "color": "#f59e0b",
  "order": 2
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "stage": {
      "_id": "stage_3",
      "name": "Review",
      "listId": "LIST_ID",
      "color": "#f59e0b",
      "order": 2
    }
  }
}
```

---

### Reorder Stages

```http
POST /api/v1/internal/stages.reorder
```

Reorder all stages within a list.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listId` | string | Yes | Parent list ID |
| `stageIds` | array | Yes | Array of stage IDs in new order |

**Request:**

```json
{
  "listId": "LIST_ID",
  "stageIds": ["stage_3", "stage_1", "stage_2"]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Stages reordered successfully"
  }
}
```

---

### Get Items in Stage

```http
GET /api/v1/internal/stages.items?stageId=STAGE_ID&limit=50
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `stageId` | string | Yes | Stage ID |
| `limit` | number | No | Max items (default: 50, max: 100) |

**Response:**

```json
{
  "success": true,
  "data": {
    "stageId": "stage_1",
    "stageName": "To Do",
    "items": [
      {
        "_id": "item_1",
        "name": "Task 1",
        "description": "Description",
        "order": 0,
        "completed": false,
        "inProgress": false,
        "createdAt": "2024-01-15T10:00:00.000Z"
      }
    ],
    "count": 1
  }
}
```

---

## Stage Color Guidelines

When creating or updating stages, use these common color codes:

| Status | Color Code | Hex |
|--------|------------|-----|
| Gray/Neutral | `#6b7280` | Gray 500 |
| Blue | `#3b82f6` | Blue 500 |
| Green | `#10b981` | Emerald 500 |
| Yellow | `#f59e0b` | Amber 500 |
| Red | `#ef4444` | Red 500 |
| Purple | `#8b5cf6` | Violet 500 |
| Pink | `#ec4899` | Pink 500 |

## Error Responses

| Error Code | Description |
|------------|-------------|
| `error-invalid-params` | Stage ID or required parameters missing |
| `error-stage-not-found` | Stage not found |
| `error-list-not-found` | List not found |
| `error-stage-has-items` | Stage has items (provide moveItemsToStageId) |
| `error-invalid-target-stage` | Target stage for moving items is invalid |

## Related Resources

- [Lists API](./lists.md) - Manage lists containing stages
- [Items API](./items.md) - Manage items within stages
