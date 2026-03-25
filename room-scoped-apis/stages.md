# Room-Scoped Stages API

## Overview

Manage workflow stages within lists in a specific room. Stages represent columns or states that items can be in (e.g., "To Do", "In Progress", "Done").

**Base Path:** `/api/v1/internal/rooms/:roomId/stages`

## Authentication

All endpoints require:
- `x-api-key`: Internal API key
- `Authorization: Bearer <JWT_TOKEN>`: Room-specific JWT token

---

## Endpoints

### Get Stages

```http
GET /api/v1/internal/rooms/:roomId/stages
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listId` | string | Yes | Parent list ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "stages": [
      {
        "_id": "STAGE_ID",
        "name": "To Do",
        "listId": "LIST_ID",
        "color": "#6b7280",
        "order": 0,
        "itemCount": 5
      },
      {
        "_id": "STAGE_ID_2",
        "name": "In Progress",
        "listId": "LIST_ID",
        "color": "#3b82f6",
        "order": 1,
        "itemCount": 3
      }
    ],
    "count": 2
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/stages?listId=LIST_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Create Stage

```http
POST /api/v1/internal/rooms/:roomId/stages
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Stage name |
| `listId` | string | Yes | Parent list ID |
| `color` | string | No | Hex color (default: #6b7280) |
| `order` | number | No | Display order (default: appended to end) |

**Response:**

```json
{
  "success": true,
  "data": {
    "stage": {
      "_id": "NEW_STAGE_ID",
      "name": "Code Review",
      "listId": "LIST_ID",
      "color": "#8b5cf6",
      "order": 2
    }
  }
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/stages" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Code Review",
    "listId": "LIST_ID",
    "color": "#8b5cf6",
    "order": 2
  }'
```

---

### Get Stage by ID

```http
GET /api/v1/internal/rooms/:roomId/stages/:stageId
```

**Response:**

```json
{
  "success": true,
  "data": {
    "stage": {
      "_id": "STAGE_ID",
      "name": "In Progress",
      "listId": "LIST_ID",
      "color": "#3b82f6",
      "order": 1
    },
    "itemCount": 3
  }
}
```

**Example:**

```bash
curl -X GET "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/stages/STAGE_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

### Update Stage

```http
PUT /api/v1/internal/rooms/:roomId/stages/:stageId
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | No | New stage name |
| `color` | string | No | New hex color |
| `order` | number | No | New display order |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Stage updated successfully",
    "stage": {
      "_id": "STAGE_ID",
      "name": "Updated Name",
      "color": "#10b981",
      "order": 1
    }
  }
}
```

**Example:**

```bash
curl -X PUT "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/stages/STAGE_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "In Review",
    "color": "#f59e0b"
  }'
```

---

### Delete Stage

```http
DELETE /api/v1/internal/rooms/:roomId/stages/:stageId
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `moveItemsToStageId` | string | Conditional | Target stage for existing items |

**Description:** Deletes a stage. If the stage contains items, you must specify a target stage to move them to.

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Stage deleted successfully",
    "deleted": true
  }
}
```

**Error (if has items and no target):**

```json
{
  "success": false,
  "error": "error-stage-has-items",
  "message": "Stage has 5 items. Provide moveItemsToStageId to move them."
}
```

**Example:**

```bash
curl -X DELETE "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/stages/STAGE_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "moveItemsToStageId": "TARGET_STAGE_ID"
  }'
```

---

## Batch Operations

### Batch Create Stages

```http
POST /api/v1/internal/rooms/:roomId/stages/batch-create
```

**Body Parameters:**

```typescript
{
  listId: string,
  stages: Array<{
    name: string,
    color?: string,
    order?: number
  }>
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch stages creation completed",
    "created": [...],
    "errors": [
      {
        "stage": { ... },
        "error": "Stage name is required"
      }
    ],
    "totalCreated": 3,
    "totalFailed": 0,
    "processingTime": 567
  }
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/stages/batch-create" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "listId": "LIST_ID",
    "stages": [
      { "name": "To Do", "color": "#6b7280", "order": 0 },
      { "name": "In Progress", "color": "#3b82f6", "order": 1 },
      { "name": "Done", "color": "#10b981", "order": 2 }
    ]
  }'
```

---

### Batch Delete Stages

```http
POST /api/v1/internal/rooms/:roomId/stages/batch-delete
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `stageIds` | array | Yes | Array of stage IDs to delete |
| `moveItemsToStageId` | string | Conditional | Target stage for existing items |

**Description:** Deletes multiple stages. All items from deleted stages are moved to the target stage.

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Batch stages deletion completed",
    "deleted": ["STAGE_ID_1", "STAGE_ID_2"],
    "errors": [...],
    "totalDeleted": 2,
    "totalFailed": 0,
    "processingTime": 1234
  }
}
```

**Example:**

```bash
curl -X POST "https://your-domain.com/api/v1/internal/rooms/ROOM_ID/stages/batch-delete" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "stageIds": ["STAGE_ID_1", "STAGE_ID_2"],
    "moveItemsToStageId": "TARGET_STAGE_ID"
  }'
```

---

## Error Codes

| Error Code | Description |
|------------|-------------|
| `error-stage-not-found` | Stage not found |
| `error-list-not-found` | List not found in this room |
| `error-invalid-target-stage` | Target stage doesn't exist or is in a different list |
| `error-stage-has-items` | Stage contains items (provide moveItemsToStageId) |
| `error-forbidden` | Stage doesn't belong to a list in this room |

---

## Stage Management Best Practices

1. **Always provide a fallback stage** when deleting stages that contain items
2. **Use consistent colors** across similar stages for better UX
3. **Order stages logically** to reflect workflow progression
4. **Avoid deleting stages** with active items when possible

## Common Stage Workflows

### Software Development

```
Backlog → In Progress → Code Review → Testing → Done
```

### Content Creation

```
Ideas → Research → Drafting → Review → Published
```

### Customer Support

```
New → Investigating → Waiting on Customer → Resolved → Closed
```
