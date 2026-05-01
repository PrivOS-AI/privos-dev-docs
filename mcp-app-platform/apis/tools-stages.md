# PrivOS MCP Tools — Stages

Stages represent kanban columns within a list.

## `privos.stages.getByList`

Get all stages for a list, sorted by order.

| | |
|---|---|
| **Scope** | `lists:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `listId` | string | Yes | List ID |

### Response

```json
[
  {
    "_id": "stage_001",
    "name": "To Do",
    "color": "#4A90D9",
    "listId": "list_001",
    "order": 0
  },
  {
    "_id": "stage_002",
    "name": "In Progress",
    "color": "#F5A623",
    "listId": "list_001",
    "order": 1
  },
  {
    "_id": "stage_003",
    "name": "Done",
    "color": "#7ED321",
    "listId": "list_001",
    "order": 2
  }
]
```

### Example

```typescript
const stages = await app.callServerTool({
  name: 'privos.stages.getByList',
  arguments: { listId: 'list_001' }
});
```

---

## `privos.stages.get`

Get a single stage by ID.

| | |
|---|---|
| **Scope** | `lists:read` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `stageId` | string | Yes | Stage ID |

### Response

```json
{
  "_id": "stage_001",
  "name": "To Do",
  "color": "#4A90D9",
  "listId": "list_001",
  "order": 0
}
```

---

## `privos.stages.create`

Create a new stage in a list.

| | |
|---|---|
| **Scope** | `lists:write` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `listId` | string | Yes | List ID |
| `name` | string | Yes | Stage name |
| `color` | string | No | Hex color (default: `#4A90D9`) |

### Response

```json
{
  "_id": "stage_004",
  "name": "Review",
  "color": "#BD10E0",
  "listId": "list_001",
  "order": 3
}
```

### Example

```typescript
await app.callServerTool({
  name: 'privos.stages.create',
  arguments: {
    listId: 'list_001',
    name: 'Review',
    color: '#BD10E0'
  }
});
```

---

## `privos.stages.update`

Update a stage's name or color.

| | |
|---|---|
| **Scope** | `lists:write` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `stageId` | string | Yes | Stage ID |
| `name` | string | No | New stage name |
| `color` | string | No | New hex color |

### Response

```json
{ "updated": true }
```

---

## `privos.stages.delete`

Delete a stage.

| | |
|---|---|
| **Scope** | `lists:write` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `stageId` | string | Yes | Stage ID |

### Response

```json
{ "deleted": true }
```

---

## `privos.stages.reorder`

Reorder all stages in a list by providing the desired order of stage IDs.

| | |
|---|---|
| **Scope** | `lists:write` |

### Arguments

| Arg | Type | Required | Description |
|-----|------|----------|-------------|
| `listId` | string | Yes | List ID |
| `stageIds` | string[] | Yes | Ordered array of stage IDs |

### Response

```json
{ "reordered": true }
```

### Example

```typescript
await app.callServerTool({
  name: 'privos.stages.reorder',
  arguments: {
    listId: 'list_001',
    stageIds: ['stage_003', 'stage_001', 'stage_002']
  }
});
```

---

## Data Model: Stage

| Field | Type | Description |
|-------|------|-------------|
| `_id` | string | Unique stage identifier |
| `name` | string | Stage display name |
| `color` | string | Hex color code |
| `listId` | string | Parent list ID |
| `order` | number | Sort order (0-based) |
