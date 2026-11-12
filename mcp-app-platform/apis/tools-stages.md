# PrivOS MCP Tools — Stages

Stages represent kanban columns within a list.

## Registered tool and authority inventory

The canonical registered names use `mcpapp.*`; `privos.*` remains a compatibility alias. Stage roots are never standalone authorities: the Stage must resolve to a List and the caller must satisfy that List's authority. App-owned Lists require an exact match with the invocation's stored six-part ownership tuple.

| Registered tool           | Scope         | Authority strategy                                 |
| ------------------------- | ------------- | -------------------------------------------------- |
| `mcpapp.stages.getByList` | `lists:read`  | List read                                          |
| `mcpapp.stages.get`       | `lists:read`  | Stage read through its List                        |
| `mcpapp.stages.create`    | `lists:write` | List write                                         |
| `mcpapp.stages.update`    | `lists:write` | Stage write through its List                       |
| `mcpapp.stages.delete`    | `lists:write` | Stage write through its List                       |
| `mcpapp.stages.reorder`   | `lists:write` | List write; every supplied Stage must belong to it |

## `privos.stages.getByList`

Get all stages for a list, sorted by order.

|           |              |
| --------- | ------------ |
| **Scope** | `lists:read` |

### Arguments

| Arg      | Type   | Required | Description |
| -------- | ------ | -------- | ----------- |
| `listId` | string | Yes      | List ID     |

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
	arguments: { listId: 'list_001' },
});
```

---

## `privos.stages.get`

Get a single stage by ID.

|           |              |
| --------- | ------------ |
| **Scope** | `lists:read` |

### Arguments

| Arg       | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| `stageId` | string | Yes      | Stage ID    |

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

Create a new stage in a list. On an app-owned list (one created by an installation bot), stage names are unique: a
create with a name that already exists on the list returns the existing stage unchanged, so retries and repeated
syncs are safe. Human lists still accept two stages with the same name.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg      | Type   | Required | Description                    |
| -------- | ------ | -------- | ------------------------------ |
| `listId` | string | Yes      | List ID                        |
| `name`   | string | Yes      | Stage name                     |
| `color`  | string | No       | Hex color (default: `#4A90D9`) |

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
		color: '#BD10E0',
	},
});
```

---

## `privos.stages.update`

Update a stage's name or color.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg       | Type   | Required | Description    |
| --------- | ------ | -------- | -------------- |
| `stageId` | string | Yes      | Stage ID       |
| `name`    | string | No       | New stage name |
| `color`   | string | No       | New hex color  |

### Response

```json
{ "updated": true }
```

---

## `privos.stages.delete`

Delete a stage. A stage that still holds items is refused; move them to another stage first.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg       | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| `stageId` | string | Yes      | Stage ID    |

### Response

```json
{ "deleted": true }
```

---

## `privos.stages.reorder`

Reorder all stages in a list by providing the desired order of stage IDs.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg        | Type     | Required | Description                |
| ---------- | -------- | -------- | -------------------------- |
| `listId`   | string   | Yes      | List ID                    |
| `stageIds` | string[] | Yes      | Ordered array of stage IDs |

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
		stageIds: ['stage_003', 'stage_001', 'stage_002'],
	},
});
```

---

## Data Model: Stage

| Field    | Type   | Description             |
| -------- | ------ | ----------------------- |
| `_id`    | string | Unique stage identifier |
| `name`   | string | Stage display name      |
| `color`  | string | Hex color code          |
| `listId` | string | Parent list ID          |
| `order`  | number | Sort order (0-based)    |
