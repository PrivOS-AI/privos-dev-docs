# Lists API

Lists are Kanban-style containers that organize items into stages. Each list can have custom field definitions and multiple stages.

## Base URL

```
/api/v1/internal/lists
```

## Endpoints

### List All Lists

```http
GET /api/v1/internal/lists.list?offset=0&count=50
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `offset` | number | No | Number of lists to skip (default: 0) |
| `count` | number | No | Number of lists to return (default: 50, max: 100) |

**Response:**

```json
{
  "success": true,
  "data": {
    "lists": [
      {
        "_id": "65a1b2c3d4e5f6g7h8i9j0k1",
        "name": "Product Backlog",
        "key": "product-backlog",
        "roomId": "ROOM_ID",
        "description": "Main product backlog",
        "templateListKey": "backlog",
        "fieldDefinitions": [
          {
            "_id": "field_1",
            "name": "Priority",
            "type": "SELECT",
            "options": ["High", "Medium", "Low"],
            "order": 0
          }
        ],
        "createdAt": "2024-01-15T10:00:00.000Z"
      }
    ],
    "count": 1,
    "offset": 0,
    "total": 10
  }
}
```

---

### Get List by ID

```http
GET /api/v1/internal/lists/:listId
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listId` | string | Yes | List ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "list": { /* List object */ },
    "stages": [
      {
        "_id": "stage_1",
        "name": "To Do",
        "color": "#6b7280",
        "order": 0,
        "templateStageKey": "todo"
      }
    ],
    "items": [
      {
        "_id": "item_1",
        "name": "Task 1",
        "stageId": "stage_1",
        "order": 0
      }
    ]
  }
}
```

---

### Update List

```http
PUT /api/v1/internal/lists/:listId
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listId` | string | Yes | List ID |

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | No | New list name |
| `description` | string | No | New list description |

**Request:**

```json
{
  "name": "Updated Backlog",
  "description": "Updated description"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "List updated successfully",
    "listId": "65a1b2c3d4e5f6g7h8i9j0k1"
  }
}
```

---

### Delete List

```http
DELETE /api/v1/internal/lists/:listId
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `listId` | string | Yes | List ID |

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "List deleted successfully"
  }
}
```

**Note:** This will also delete all associated items and stages.

---

### Get Lists by Template Key

```http
GET /api/v1/internal/lists.byTemplateKey?roomId=ROOM_ID&templateKey=backlog
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `templateKey` | string | Yes | Template key |

**Response:**

```json
{
  "success": true,
  "data": {
    "lists": [
      {
        "_id": "list_1",
        "name": "Backlog",
        "templateListKey": "backlog",
        "stages": [...],
        "items": [...]
      }
    ]
  }
}
```

---

### Get List by Template List Key

```http
GET /api/v1/internal/lists.byTemplateListKey?roomId=ROOM_ID&templateListKey=sprint-backlog
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `roomId` | string | Yes | Room ID |
| `templateListKey` | string | Yes | Template list key |

**Response:**

```json
{
  "success": true,
  "data": {
    "list": { /* List object */ },
    "stages": [...],
    "items": [...]
  }
}
```

---

### Get Lists by Room ID

```http
GET /api/v1/internal/lists.byRoomId?roomId=ROOM_ID&offset=0&count=50
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
    "lists": [
      {
        "_id": "list_1",
        "name": "Backlog",
        "description": "...",
        "createdAt": "2024-01-15T10:00:00.000Z",
        "stageCount": 4,
        "itemCount": 25
      }
    ],
    "count": 1,
    "offset": 0,
    "total": 5
  }
}
```

---

### Create List

```http
POST /api/v1/internal/lists.create
```

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `_id` | string | No | Caller-supplied list `_id`. Must be unique. Auto-generated (17-char Meteor ID) if omitted. |
| `name` | string | Yes | List name |
| `roomId` | string | Yes | Room ID where list belongs |
| `description` | string | No | List description |
| `fieldDefinitions` | array | No | Custom field definitions |
| `stages` | array | No | Initial stages to create |

**Request:**

```json
{
  "name": "Sprint Backlog",
  "roomId": "ROOM_ID",
  "description": "Sprint 24 backlog",
  "fieldDefinitions": [
    {
      "_id": "field_priority",
      "name": "Priority",
      "type": "SELECT",
      "options": ["High", "Medium", "Low"],
      "order": 0
    },
    {
      "_id": "field_assignee",
      "name": "Assignee",
      "type": "ASSIGNEE",
      "order": 1
    }
  ],
  "stages": [
    {
      "name": "To Do",
      "color": "#6b7280",
      "order": 0
    },
    {
      "name": "In Progress",
      "color": "#3b82f6",
      "order": 1
    },
    {
      "name": "Done",
      "color": "#10b981",
      "order": 2
    }
  ]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "list": {
      "_id": "65a1b2c3d4e5f6g7h8i9j0k1",
      "name": "Sprint Backlog",
      "roomId": "ROOM_ID"
    }
  }
}
```

**Field Definition Types:**

Full enum (`server/core-typings/IList.ts`): `TEXT`, `TEXTAREA`, `NUMBER`, `DATE`,
`DATE_TIME`, `SELECT`, `MULTI_SELECT`, `USER`, `CHECKBOX`, `URL`, `FILE`,
`FILE_MULTIPLE`, `DOCUMENT`, `ASSIGNEE`, `DEADLINE`, `DEPENDENCIES`, `RECURRING`.
There is no `USER_SELECT` or `MEMBER_SELECT` type.

| Type | Description | Example Value |
|------|-------------|---------------|
| `TEXT` | Plain text input | `"Sample text"` |
| `TEXTAREA` | Multi-line text | `"Line 1\nLine 2"` |
| `NUMBER` | Numeric input | `42` |
| `DATE` | Date picker | `"2024-01-15"` |
| `DATE_TIME` | Date + time picker | `"2024-01-15T10:00:00.000Z"` |
| `SELECT` | Single select from options | `"High"` |
| `MULTI_SELECT` | Multiple select | `["A", "B"]` |
| `USER` | Single user reference | `"USER_ID"` |
| `CHECKBOX` | Boolean checkbox | `true` |
| `URL` | Link | `"https://example.com"` |
| `FILE` | Single file reference | file ref object |
| `FILE_MULTIPLE` | Multiple file references | array of file ref objects |
| `DOCUMENT` | Rich text document | `[{"_id": "...", "content": "..."}]` |
| `ASSIGNEE` | User(s) assigned to the item — see [Assignee values](#assignee-field-values) below | `"USER_ID"`, `{"_id": "USER_ID"}`, or an array of either |
| `DEADLINE` | Due-date style field | `"2024-01-15"` |
| `DEPENDENCIES` | Links to other items | array of item IDs |
| `RECURRING` | Recurrence rule | recurrence config object |

#### Assignee field values

`ASSIGNEE` fields accept a bare user ID, a single `{ _id, username? }` object, or
an array of either — all three shapes resolve to the same set of assigned user
IDs (`app/api/server/lib/isolated-list-item-filter.ts:23-41`).

#### Isolated lists

A list with `isolatedList: true` restricts item visibility: only the room
owner/admin, the item's creator, and any user in an `ASSIGNEE` field on that item
(plus their visible sub-items) can see it — everyone else sees nothing
(`app/api/server/lib/isolated-list-item-filter.ts:65-93,99-128`). The `ASSIGNEE`
field is what drives this — assigning a user is how you grant them visibility
into an otherwise isolated item.

---

### Manage Field Definitions

```http
GET /api/v1/internal/lists.fields/:listId
```

Get all field definitions for a list.

**Response:**

```json
{
  "success": true,
  "data": {
    "fieldDefinitions": [
      {
        "_id": "field_1",
        "name": "Priority",
        "type": "SELECT",
        "options": ["High", "Medium", "Low"],
        "order": 0
      }
    ]
  }
}
```

---

```http
POST /api/v1/internal/lists.fields/:listId
```

Add a new field definition to a list.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `field` | object | Yes | Field definition |

**Request:**

```json
{
  "field": {
    "_id": "field_status",
    "name": "Status",
    "type": "SELECT",
    "options": ["Open", "Closed"],
    "order": 10
  }
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Field added successfully",
    "fieldId": "field_status"
  }
}
```

---

```http
PUT /api/v1/internal/lists.fields/:listId
```

Update all field definitions for a list (replaces entire array).

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fieldDefinitions` | array | Yes | Array of field definitions |

**Request:**

```json
{
  "fieldDefinitions": [
    {
      "_id": "field_1",
      "name": "Priority",
      "type": "SELECT",
      "options": ["High", "Medium", "Low"],
      "order": 0
    },
    {
      "_id": "field_2",
      "name": "Assignee",
      "type": "ASSIGNEE",
      "order": 1
    }
  ]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "message": "Field definitions updated successfully"
  }
}
```

## Error Responses

| Error Code | Description |
|------------|-------------|
| `error-invalid-params` | List ID is required or invalid (e.g. `_id` is not a non-empty string) |
| `error-list-not-found` | List not found |
| `error-invalid-field` | Field definition not found in list |
| `error-duplicate-id` | A list with the supplied `_id` already exists |

## MCP App Access (`mcpapp.lists.*`)

MCP apps reach lists through a separate MCP tool surface
(`server/services/mcp-tool-handlers-lists.ts`), not this internal REST API —
`mcpapp.lists.getAll`/`get` require `lists:read`; `mcpapp.lists.create`,
`updateItem`, `updateCustomField`, etc. require `lists:write`.

- `mcpapp.lists.create` accepts an `isolatedList: boolean` argument and
  **requires the caller to be a room owner or admin** — enforced at the tool
  handler (`mcp-tool-handlers-lists.ts` around lines 59-78), independent of the
  `lists:write` scope check.
- Assigning a user to an item (setting an `ASSIGNEE` field, which is what grants
  visibility on an isolated list — see [Isolated lists](#isolated-lists) above)
  goes through `mcpapp.lists.updateItem` or `mcpapp.lists.updateCustomField` and
  only needs the ordinary `lists:write` scope; no extra room-role check applies
  to assignment itself.

## Related Resources

- [Items API](./items.md) - Manage items within lists
- [Stages API](./stages.md) - Manage stages within lists
