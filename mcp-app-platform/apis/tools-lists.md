# PrivOS MCP Tools — Lists & Items

## Registered tool and authority inventory

The canonical registered names use `mcpapp.*`. The legacy `privos.*` spelling remains an alias for compatibility. Every object-rooted call first resolves its List and applies the authority strategy below; Item and Stage roots must belong to that authorized List. For app-owned Lists, the authorized invocation's exact six-part ownership tuple must match the stored tuple. Collection and batch tools apply the same rule to every result or target before returning data or mutating anything.

| Registered tool                  | Scope         | Authority strategy                                                 |
| -------------------------------- | ------------- | ------------------------------------------------------------------ |
| `mcpapp.lists.getAll`            | `lists:read`  | Authorized Room collection; filters every List                     |
| `mcpapp.lists.get`               | `lists:read`  | List read                                                          |
| `mcpapp.lists.create`            | `lists:write` | Authorized Room create; V3 app calls stamp exact ownership         |
| `mcpapp.lists.getItems`          | `lists:read`  | List read plus isolated Item visibility                            |
| `mcpapp.lists.queryItems`        | `lists:query` | List query plus isolated Item visibility                           |
| `mcpapp.lists.createItem`        | `lists:write` | List write; Stage and dependency references validated first        |
| `mcpapp.lists.updateItem`        | `lists:write` | Item write through its List                                        |
| `mcpapp.lists.addField`          | `lists:write` | List write                                                         |
| `mcpapp.lists.updateField`       | `lists:write` | List write                                                         |
| `mcpapp.lists.addFieldOption`    | `lists:write` | List write                                                         |
| `mcpapp.lists.updateFieldOption` | `lists:write` | List write                                                         |
| `mcpapp.lists.removeFieldOption` | `lists:write` | List write                                                         |
| `mcpapp.lists.reorderFields`     | `lists:write` | List write; the field set must match the list exactly              |
| `mcpapp.lists.deleteItem`        | `lists:write` | Item write through its List                                        |
| `mcpapp.lists.deleteItems`       | `lists:write` | All Item/List targets authorized before one delete                 |
| `mcpapp.lists.getItem`           | `lists:read`  | Item read through its List                                         |
| `mcpapp.lists.getItemsByStage`   | `lists:read`  | List read; Stage must belong to List                               |
| `mcpapp.lists.getItemsByStages`  | `lists:read`  | List read; every Stage must belong to List                         |
| `mcpapp.lists.moveItemToStage`   | `lists:write` | Item write; destination Stage must belong to List                  |
| `mcpapp.lists.reorderItem`       | `lists:write` | Item write through its List and current Stage                      |
| `mcpapp.lists.updateCustomField` | `lists:write` | Item write; dependency references use invocation authority         |
| `mcpapp.lists.patchItemFields`   | `lists:write` | Item write; dependency references use invocation authority         |
| `mcpapp.lists.searchItems`       | `lists:read`  | List read plus isolated Item visibility                            |
| `mcpapp.lists.getSubItems`       | `lists:read`  | Parent Item read plus descendant visibility                        |
| `mcpapp.lists.removeField`       | `lists:write` | List write                                                         |
| `mcpapp.lists.updateList`        | `lists:write` | List write; app-owned isolation cannot be removed                  |
| `mcpapp.lists.delete`            | `lists:write` | List write                                                         |
| `mcpapp.lists.deleteMany`        | `lists:write` | Every List authorized before the first delete                      |
| `mcpapp.lists.batchAddFields`    | `lists:write` | List write                                                         |
| `mcpapp.lists.batchCreateItems`  | `lists:write` | Entire Stage/dependency set authorized before the first Item write |
| `mcpapp.lists.getTimeline`       | `lists:read`  | List read plus isolated Item visibility; foreign refs label-only   |
| `mcpapp.lists.shiftItems`        | `lists:write` | List write; every target passes the `items.patchFields` item gates |

### `lists:query`

`mcpapp.lists.queryItems` declares and enforces only `lists:query`; it does not require the separate `lists:read` grant. The scope permits structured filters, caller-selected response fields, stable keyset pagination, and optional projection of dependency fields explicitly shared into the queried List. It does not authorize any mutation, does not bypass isolated descendant visibility, and does not weaken exact app-ownership or Room-binding checks. A query authorized under one binding cannot select an app-owned List from another binding or Room.

## `privos.lists.create`

Create a new list in a room with optional field definitions and kanban stages.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg                          | Type   | Required | Description                                                      |
| ---------------------------- | ------ | -------- | ---------------------------------------------------------------- |
| `roomId`                     | string | Yes      | Room ID to create the list in                                    |
| `name`                       | string | No       | List name (default: "New List")                                  |
| `key`                        | string | No       | List key, must be unique (auto-generated from name if omitted)   |
| `description`                | string | No       | List description                                                 |
| `templateKey`                | string | No       | Stable adoption anchor, 1-64 of `A-Z a-z 0-9 . _ : -`; see below |
| `fieldDefinitions`           | array  | No       | Custom field definitions                                         |
| `fieldDefinitions[]._id`     | string | No       | Field ID (auto-generated if omitted)                             |
| `fieldDefinitions[].name`    | string | Yes      | Field label                                                      |
| `fieldDefinitions[].type`    | enum   | Yes      | Field type (see `addField` for full list)                        |
| `fieldDefinitions[].options` | array  | No       | For SELECT/MULTI_SELECT: `[{ _id?, value, color? }]`             |
| `stages`                     | array  | No       | Kanban stages (defaults to single "Ungroup" stage)               |
| `stages[].name`              | string | Yes      | Stage name                                                       |
| `stages[].color`             | string | No       | Hex color (default: #6b7280)                                     |

### Response

```json
{
  "list": {
    "_id": "list_abc",
    "name": "CRM Pipeline",
    "key": "CRMP",
    "roomId": "room_xyz789",
    "fieldDefinitions": [
      { "_id": "f1", "name": "Email", "type": "TEXT", "order": 0 },
      { "_id": "f2", "name": "Status", "type": "SELECT", "order": 1, "options": [...] }
    ],
    "createdAt": "2026-04-04T10:00:00Z"
  },
  "stages": [
    { "_id": "stage_001", "name": "New", "color": "#3b82f6", "order": 0 },
    { "_id": "stage_002", "name": "In Progress", "color": "#f59e0b", "order": 1 },
    { "_id": "stage_003", "name": "Done", "color": "#22c55e", "order": 2 }
  ]
}
```

### `templateKey`: one list per Room

`templateKey` is persisted on the List and returned by every List read (`lists.get`,
`lists.getAll`, REST `lists.info`, `lists.list`, `lists.listByRoomId`). A Room holds at most one
live List per `templateKey` through the create paths: when the Room already has one, this tool
and REST `lists.create` return that List with `existing: true` (REST: `{ list, defaultStage,
existing: true }`) and create nothing; the other arguments are ignored. The create is refused with
`error-duplicate-template-key` when the caller may not write the existing List (for example
another app's List) or names a different `_id`.

Because app-created Lists are hidden by default, an app bootstrapping through REST looks its List
up with `lists.list?roomId=<roomId>&templateKey=<key>` (or `lists.listByRoomId` with `templateKey`):
a `templateKey` lookup includes hidden Lists; the normal access rules still apply.

### Example

```typescript
// Create a list with fields and stages
await app.callServerTool({
	name: 'privos.lists.create',
	arguments: {
		roomId: 'room_xyz789',
		name: 'CRM Pipeline',
		description: 'Track customer leads',
		fieldDefinitions: [
			{ name: 'Email', type: 'TEXT' },
			{
				name: 'Priority',
				type: 'SELECT',
				options: [
					{ value: 'High', color: '#e74c3c' },
					{ value: 'Low', color: '#2ecc71' },
				],
			},
			{ name: 'Assignee', type: 'ASSIGNEE' },
		],
		stages: [
			{ name: 'New', color: '#3b82f6' },
			{ name: 'In Progress', color: '#f59e0b' },
			{ name: 'Done', color: '#22c55e' },
		],
	},
});

// Minimal — creates list with default "Ungroup" stage
await app.callServerTool({
	name: 'privos.lists.create',
	arguments: { roomId: 'room_xyz789', name: 'Tasks' },
});
```

---

## `privos.lists.getAll`

Get all lists in a room.

|           |              |
| --------- | ------------ |
| **Scope** | `lists:read` |

### Arguments

| Arg      | Type   | Required | Description |
| -------- | ------ | -------- | ----------- |
| `roomId` | string | Yes      | Room ID     |

### Response

```json
[
	{
		"_id": "list_001",
		"name": "Contacts",
		"description": "Customer contact list",
		"roomId": "room_xyz789",
		"fieldDefinitions": [
			{ "_id": "field_001", "name": "Email", "type": "TEXT", "order": 0 },
			{ "_id": "field_002", "name": "Status", "type": "SELECT", "order": 1, "options": [{ "value": "Active" }, { "value": "Inactive" }] }
		],
		"createdAt": "2026-03-25T10:00:00Z"
	}
]
```

### Example

```typescript
await app.callServerTool({
	name: 'privos.lists.getAll',
	arguments: { roomId: 'room_xyz789' },
});
```

---

## `privos.lists.get`

Get a single list by ID or key, including field definitions.

|           |              |
| --------- | ------------ |
| **Scope** | `lists:read` |

### Arguments

At least one of `listId` or `key` is required.

| Arg      | Type   | Required | Description                       |
| -------- | ------ | -------- | --------------------------------- |
| `listId` | string | No       | List ID                           |
| `key`    | string | No       | List key (e.g. `"CRM"`, `"TASK"`) |

### Response

```json
{
	"_id": "list_001",
	"name": "Contacts",
	"description": "Customer contact list",
	"roomId": "room_xyz789",
	"fieldDefinitions": [
		{ "_id": "field_001", "name": "Email", "type": "TEXT", "order": 0 },
		{ "_id": "field_002", "name": "Phone", "type": "TEXT", "order": 1 }
	],
	"createdAt": "2026-03-25T10:00:00Z"
}
```

---

## `privos.lists.updateList`

Update a list's name, description, isolated mode or visibility. Only the properties sent change.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg            | Type    | Required | Description                                                                  |
| -------------- | ------- | -------- | ---------------------------------------------------------------------------- |
| `listId`       | string  | Yes      | List ID                                                                      |
| `name`         | string  | No       | New list name                                                                |
| `description`  | string  | No       | New list description                                                         |
| `isolatedList` | boolean | No       | Isolated mode; needs a room owner/admin, and app-owned lists cannot leave it |
| `hidden`       | boolean | No       | Hide the list from the room list panel for ordinary members; changing it needs a room owner/admin |

### Response

```json
{ "updated": true }
```

---

## `privos.lists.delete`

Soft-delete a list by ID. Sets `_deletedAt` timestamp instead of permanently removing the list. Items and stages cascade-delete but can be recovered if the list is recovered.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg      | Type   | Required | Description       |
| -------- | ------ | -------- | ----------------- |
| `listId` | string | Yes      | List ID to delete |

### Response

```json
{ "deleted": true, "listId": "list_001" }
```

### Example

```typescript
await app.callServerTool({
	name: 'privos.lists.delete',
	arguments: { listId: 'list_001' },
});
```

---

## `privos.lists.deleteMany`

Soft-delete multiple lists by IDs in one call. Maximum 100 lists per batch. Every List is resolved and authorized before the first delete; an invalid or unauthorized target rejects the batch without deleting any List.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg       | Type     | Required | Description                           |
| --------- | -------- | -------- | ------------------------------------- |
| `listIds` | string[] | Yes      | Array of list IDs to delete (max 100) |

### Response

```json
{
	"deleted": true,
	"deletedCount": 3,
	"listIds": ["list_001", "list_002", "list_003"]
}
```

### Example

```typescript
await app.callServerTool({
	name: 'privos.lists.deleteMany',
	arguments: {
		listIds: ['list_001', 'list_002', 'list_003'],
	},
});
```

---

## `privos.lists.getItems`

Get items in a list with pagination, sorting, and optional stage filter.

|           |              |
| --------- | ------------ |
| **Scope** | `lists:read` |

### Arguments

| Arg                                      | Type   | Required | Description                                                     |
| ---------------------------------------- | ------ | -------- | --------------------------------------------------------------- |
| `listId`                                 | string | Yes      | List ID                                                         |
| `offset`                                 | number | No       | Position to start from (default: 0)                             |
| `count`                                  | number | No       | Number of items to return (default: 50, max: 100)               |
| `sortBy`                                 | enum   | No       | Sort field: `createdAt`, `name`, `order` (default: `createdAt`) |
| `sortOrder`                              | enum   | No       | Sort direction: `asc`, `desc` (default: `desc`)                 |
| `stageId`                                | string | No       | Filter by stage ID                                              |
| `customFieldFilters`                     | array  | No       | Filter by custom field values                                   |
| `customFieldFilters[].fieldDefinitionId` | string | Yes      | Field definition ID                                             |
| `customFieldFilters[].operator`          | enum   | Yes      | `equal` or `in`                                                 |
| `customFieldFilters[].value`             | any    | Yes      | Single value for `equal`, array for `in`                        |

### Response

```json
{
	"items": [
		{
			"_id": "item_001",
			"name": "John Doe",
			"description": "VIP customer",
			"listId": "list_001",
			"stageId": "stage_001",
			"order": 0,
			"customFields": [
				{ "fieldId": "field_001", "value": "john@example.com" },
				{ "fieldId": "field_002", "value": "+1-555-0100" }
			],
			"createdAt": "2026-03-25T10:00:00Z"
		}
	],
	"count": 50,
	"offset": 0,
	"total": 150
}
```

### Example

```typescript
// First page, newest first
await app.callServerTool({
	name: 'privos.lists.getItems',
	arguments: { listId: 'list_001' },
});

// Second page
await app.callServerTool({
	name: 'privos.lists.getItems',
	arguments: { listId: 'list_001', offset: 50, count: 50 },
});

// Filter by stage, sorted by order
await app.callServerTool({
	name: 'privos.lists.getItems',
	arguments: { listId: 'list_001', stageId: 'stage_001', sortBy: 'order', sortOrder: 'asc' },
});

// Filter by custom fields
await app.callServerTool({
	name: 'privos.lists.getItems',
	arguments: {
		listId: 'list_001',
		customFieldFilters: [
			{ fieldDefinitionId: 'field_status', operator: 'equal', value: 'Active' },
			{ fieldDefinitionId: 'field_priority', operator: 'in', value: ['High', 'Critical'] },
		],
	},
});
```

---

## `privos.lists.getItem`

Get full item detail by ID, including all custom fields.

|           |              |
| --------- | ------------ |
| **Scope** | `lists:read` |

### Arguments

| Arg      | Type   | Required | Description |
| -------- | ------ | -------- | ----------- |
| `itemId` | string | Yes      | Item ID     |

### Response

```json
{
	"_id": "item_001",
	"name": "John Doe",
	"description": "VIP customer",
	"listId": "list_001",
	"stageId": "stage_001",
	"order": 0,
	"customFields": {
		"field_001": "john@example.com",
		"field_002": "+1-555-0100"
	},
	"parentId": null,
	"createdAt": "2026-03-25T10:00:00Z"
}
```

---

## `privos.lists.createItem`

Create a new item in a list, optionally in a given stage or as a sub-item of another item.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg            | Type   | Required | Description                                                              |
| -------------- | ------ | -------- | ------------------------------------------------------------------------ |
| `listId`       | string | Yes      | List ID                                                                  |
| `title`        | string | Yes      | Item title/name                                                          |
| `description`  | string | No       | Item description                                                         |
| `stageId`      | string | No       | Stage ID in this list; default: the parent's stage, else the first stage |
| `parentId`     | string | No       | Create as a sub-item of this item (same list, at most 10 levels deep)    |
| `customFields` | array  | No       | Array of `{ fieldId: string, value: any }`                               |

A sub-item is placed after its existing siblings. Read sub-items with `getSubItems`.

### Response

```json
{
	"_id": "item_123",
	"name": "John Doe",
	"listId": "list_001"
}
```

### Example

```typescript
await app.callServerTool({
	name: 'privos.lists.createItem',
	arguments: {
		listId: 'list_001',
		title: 'John Doe',
		description: 'New customer',
		customFields: [
			{ fieldId: 'field_001', value: 'john@example.com' },
			{ fieldId: 'field_002', value: '+1-555-0100' },
		],
	},
});
```

---

## `privos.lists.updateItem`

Update an item's title, description, or custom fields.

`customFields` **replaces the whole array**: a field you do not send loses its value. To change only some fields,
use `patchItemFields`. A refused field value changes nothing, title and description included.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg            | Type   | Required | Description                                |
| -------------- | ------ | -------- | ------------------------------------------ |
| `itemId`       | string | Yes      | Item ID                                    |
| `title`        | string | No       | New item title                             |
| `description`  | string | No       | New item description                       |
| `customFields` | array  | No       | Array of `{ fieldId: string, value: any }` |

### Response

```json
{ "updated": true }
```

---

## `privos.lists.deleteItem`

Delete an item from a list.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg      | Type   | Required | Description |
| -------- | ------ | -------- | ----------- |
| `itemId` | string | Yes      | Item ID     |

### Response

```json
{ "deleted": true }
```

---

## `privos.lists.deleteItems`

Batch delete multiple items by IDs or by custom field filter query. At least one of `itemIds` or `customFieldFilters` is required.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg                                      | Type     | Required    | Description                              |
| ---------------------------------------- | -------- | ----------- | ---------------------------------------- |
| `listId`                                 | string   | Conditional | Required when using `customFieldFilters` |
| `itemIds`                                | string[] | No          | Array of item IDs to delete              |
| `customFieldFilters`                     | array    | No          | Delete items matching these conditions   |
| `customFieldFilters[].fieldDefinitionId` | string   | Yes         | Field definition ID                      |
| `customFieldFilters[].operator`          | enum     | Yes         | `equal` or `in`                          |
| `customFieldFilters[].value`             | any      | Yes         | Single value for `equal`, array for `in` |

### Response

```json
{ "deleted": true, "deletedCount": 5 }
```

### Example

```typescript
// Delete by IDs
await app.callServerTool({
	name: 'privos.lists.deleteItems',
	arguments: {
		itemIds: ['item_001', 'item_002', 'item_003'],
	},
});

// Delete by custom field filter
await app.callServerTool({
	name: 'privos.lists.deleteItems',
	arguments: {
		listId: 'list_001',
		customFieldFilters: [{ fieldDefinitionId: 'field_status', operator: 'equal', value: 'Archived' }],
	},
});

// Delete by multiple conditions
await app.callServerTool({
	name: 'privos.lists.deleteItems',
	arguments: {
		listId: 'list_001',
		customFieldFilters: [
			{ fieldDefinitionId: 'field_status', operator: 'in', value: ['Cancelled', 'Expired'] },
			{ fieldDefinitionId: 'field_priority', operator: 'equal', value: 'Low' },
		],
	},
});
```

---

## `privos.lists.getItemsByStage`

Get items in a specific kanban stage.

|           |              |
| --------- | ------------ |
| **Scope** | `lists:read` |

### Arguments

| Arg       | Type   | Required | Description         |
| --------- | ------ | -------- | ------------------- |
| `listId`  | string | Yes      | List ID             |
| `stageId` | string | Yes      | Stage ID            |
| `limit`   | number | No       | Max items to return |

### Response

```json
[
  {
    "_id": "item_001",
    "name": "John Doe",
    "listId": "list_001",
    "stageId": "stage_001",
    "order": 0,
    "customFields": { ... }
  }
]
```

---

## `privos.lists.getItemsByStages`

Get all items grouped by multiple stages in one call. Useful for loading a kanban board or viewing items across several columns at once.

|           |              |
| --------- | ------------ |
| **Scope** | `lists:read` |

### Arguments

| Arg        | Type     | Required | Description                 |
| ---------- | -------- | -------- | --------------------------- |
| `listId`   | string   | Yes      | List ID                     |
| `stageIds` | string[] | Yes      | Array of stage IDs to fetch |

### Response

Items grouped by stage ID. Stages with no items return an empty array.

```json
{
  "stage_001": [
    { "_id": "item_001", "name": "Task A", "stageId": "stage_001", "order": 0, "customFields": [...] },
    { "_id": "item_002", "name": "Task B", "stageId": "stage_001", "order": 1, "customFields": [...] }
  ],
  "stage_002": [
    { "_id": "item_003", "name": "Task C", "stageId": "stage_002", "order": 0, "customFields": [...] }
  ],
  "stage_003": []
}
```

### Example

```typescript
// Load all items for 3 kanban columns
await app.callServerTool({
	name: 'privos.lists.getItemsByStages',
	arguments: {
		listId: 'list_001',
		stageIds: ['stage_001', 'stage_002', 'stage_003'],
	},
});
```

---

## `privos.lists.moveItemToStage`

Move an item to a different kanban stage.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg       | Type   | Required | Description     |
| --------- | ------ | -------- | --------------- |
| `itemId`  | string | Yes      | Item ID         |
| `stageId` | string | Yes      | Target stage ID |

### Response

```json
{ "moved": true }
```

---

## `privos.lists.reorderItem`

Change an item's position within its current stage.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg      | Type   | Required | Description        |
| -------- | ------ | -------- | ------------------ |
| `itemId` | string | Yes      | Item ID            |
| `order`  | number | Yes      | New order position |

### Response

```json
{ "reordered": true }
```

---

## `privos.lists.updateCustomField`

Update a single custom field value on an item.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg       | Type   | Required | Description         |
| --------- | ------ | -------- | ------------------- |
| `itemId`  | string | Yes      | Item ID             |
| `fieldId` | string | Yes      | Field definition ID |
| `value`   | any    | Yes      | New field value     |

### Response

```json
{ "updated": true }
```

### Example

```typescript
await app.callServerTool({
	name: 'privos.lists.updateCustomField',
	arguments: {
		itemId: 'item_001',
		fieldId: 'field_002',
		value: '+1-555-0200',
	},
});
```

---

## `privos.lists.searchItems`

Search items by name in a list.

|           |              |
| --------- | ------------ |
| **Scope** | `lists:read` |

### Arguments

| Arg      | Type   | Required | Description               |
| -------- | ------ | -------- | ------------------------- |
| `listId` | string | Yes      | List ID                   |
| `query`  | string | Yes      | Search query              |
| `limit`  | number | No       | Max results (default: 50) |

### Response

```json
[
  {
    "_id": "item_001",
    "name": "John Doe",
    "listId": "list_001",
    "customFields": { ... }
  }
]
```

---

## `privos.lists.getSubItems`

Get child/sub-items of a parent item.

|           |              |
| --------- | ------------ |
| **Scope** | `lists:read` |

### Arguments

| Arg        | Type   | Required | Description    |
| ---------- | ------ | -------- | -------------- |
| `parentId` | string | Yes      | Parent item ID |

### Response

```json
[
	{
		"_id": "item_002",
		"name": "Sub-task 1",
		"parentId": "item_001",
		"listId": "list_001"
	}
]
```

---

## `privos.lists.addField`

Add a new custom field definition to a list.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg       | Type   | Required | Description                                          |
| --------- | ------ | -------- | ---------------------------------------------------- |
| `listId`  | string | Yes      | List ID                                              |
| `fieldId` | string | No       | Custom field ID (auto-generated if omitted)          |
| `name`    | string | Yes      | Field label                                          |
| `type`    | enum   | Yes      | Field type (see below)                               |
| `options` | array  | No       | For SELECT/MULTI_SELECT: `[{ _id?, value, color? }]` |

**Field Types:** `TEXT`, `TEXTAREA`, `NUMBER`, `DATE`, `DATE_TIME`, `SELECT`, `MULTI_SELECT`, `CHECKBOX`, `URL`, `USER`, `FILE`, `FILE_MULTIPLE`, `DOCUMENT`, `ASSIGNEE`, `DEADLINE`, `DEPENDENCIES`

### Response

```json
{
	"_id": "field_789",
	"name": "Email",
	"type": "TEXT",
	"order": 3
}
```

### Example: SELECT field with options

```typescript
await app.callServerTool({
	name: 'privos.lists.addField',
	arguments: {
		listId: 'list_001',
		name: 'Source',
		type: 'SELECT',
		options: [{ value: 'Web' }, { value: 'Email' }, { value: 'Phone' }],
	},
});
```

---

## `privos.lists.removeField`

Remove a custom field definition from a list.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg       | Type   | Required | Description                   |
| --------- | ------ | -------- | ----------------------------- |
| `listId`  | string | Yes      | List ID                       |
| `fieldId` | string | Yes      | Field definition ID to remove |

### Response

```json
{ "removed": true }
```

---

## SELECT and MULTI_SELECT values

A SELECT value is a string and a MULTI_SELECT value is an array of strings. Hub stores them exactly as sent and does
not check them against the field's options, so an app reads back what it wrote. The Hub UI shows a value only when it
is an option `_id`; any other string is stored but shows as empty. Hub logs a warning for every write of a value that
is not an option `_id`.

To have values that both your app and the UI understand, give every option a stable `_id` that is your app's own code
(for example `{ "_id": "active", "value": "Active" }`) and write that `_id`. For lists that already exist, add the
options with `addFieldOption` or `updateField`; values already stored as those codes then show in the UI with no item
rewrite. Do not use `removeField` plus `addField` for this: `removeField` deletes the field's value from every item.

---

## `privos.lists.updateField`

Change a custom field definition in place. Only the properties present in `data` change, and item values are kept.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg                  | Type   | Required | Description                                                                |
| -------------------- | ------ | -------- | -------------------------------------------------------------------------- |
| `listId`             | string | Yes      | List ID                                                                    |
| `fieldId`            | string | Yes      | Field definition ID                                                        |
| `data.name`          | string | No       | New label                                                                  |
| `data.type`          | enum   | No       | New type (see `addField`); a field cannot change to or from `DEPENDENCIES` |
| `data.options`       | array  | No       | Replaces the whole option list: `[{ _id?, value, color? }]`                |
| `data.order`         | number | No       | Position among the list's fields                                           |
| `data.displayFormat` | object | No       | Display format config                                                      |

Changing the type keeps stored item values as they are and does not convert them (an app moving a SELECT of codes to
TEXT keeps every value). A `null` property counts as not sent. Option ids must be unique within the field. Sending `options` replaces every option; to change one option while
other writers may be editing the field, use the option tools below.

### Response

```json
{ "field": { "_id": "f_status", "name": "Status", "type": "TEXT", "order": 2 } }
```

---

## `privos.lists.addFieldOption`

Add one option to a SELECT or MULTI_SELECT field.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg       | Type   | Required | Description                                                          |
| --------- | ------ | -------- | -------------------------------------------------------------------- |
| `listId`  | string | Yes      | List ID                                                              |
| `fieldId` | string | Yes      | SELECT or MULTI_SELECT field ID                                      |
| `_id`     | string | No       | Option ID, for example your app's code; must be unused in this field |
| `value`   | string | Yes      | Label shown in the UI                                                |
| `color`   | string | No       | Hex color                                                            |

Adding an `_id` that the field already has fails with `Option <id> already exists`, so the call is safe to retry.

### Response

```json
{ "option": { "_id": "active", "value": "Active", "order": 0 } }
```

---

## `privos.lists.updateFieldOption`

Change the label, color or order of one option. The option ID never changes.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg        | Type   | Required | Description                     |
| ---------- | ------ | -------- | ------------------------------- |
| `listId`   | string | Yes      | List ID                         |
| `fieldId`  | string | Yes      | SELECT or MULTI_SELECT field ID |
| `optionId` | string | Yes      | Option ID                       |
| `value`    | string | No       | New label                       |
| `color`    | string | No       | New hex color                   |
| `order`    | number | No       | New position                    |

### Response

```json
{ "updated": true }
```

---

## `privos.lists.removeFieldOption`

Remove one option from a SELECT or MULTI_SELECT field. Items that hold the option ID keep it as a plain value.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg        | Type   | Required | Description                     |
| ---------- | ------ | -------- | ------------------------------- |
| `listId`   | string | Yes      | List ID                         |
| `fieldId`  | string | Yes      | SELECT or MULTI_SELECT field ID |
| `optionId` | string | Yes      | Option ID                       |

### Response

```json
{ "removed": true }
```

---

## `privos.lists.reorderFields`

Set the order of a list's custom fields in one write. Nothing else about the fields changes.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg        | Type     | Required | Description                                               |
| ---------- | -------- | -------- | --------------------------------------------------------- |
| `listId`   | string   | Yes      | List ID                                                   |
| `fieldIds` | string[] | Yes      | Every field ID of the list exactly once, in the new order |

If a field was added or removed between your read and this call, the call fails with
`The list fields changed while reordering`; read the list again and retry.

### Response

```json
{ "reordered": true }
```

---

## `privos.lists.patchItemFields`

Set several custom field values on one item. Only the fields sent change; every other field keeps its value. Send
`null` as the value to clear a field. Use this instead of `updateItem` when you change only some fields:
`updateItem.customFields` replaces the whole array.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg            | Type   | Required | Description                       |
| -------------- | ------ | -------- | --------------------------------- |
| `itemId`       | string | Yes      | Item ID                           |
| `customFields` | array  | Yes      | Non-empty: `[{ fieldId, value }]` |

Values are checked the same way as on create (see "SELECT and MULTI_SELECT values" above); a refused value
changes nothing.

### Response

```json
{ "updated": true }
```

---

## `privos.lists.batchAddFields`

Add multiple custom field definitions to a list in one call. More efficient than calling `addField` repeatedly.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg                | Type   | Required | Description                                          |
| ------------------ | ------ | -------- | ---------------------------------------------------- |
| `listId`           | string | Yes      | Target list ID                                       |
| `fields`           | array  | Yes      | Array of field definitions to create                 |
| `fields[]._id`     | string | No       | Field ID (auto-generated if omitted)                 |
| `fields[].name`    | string | Yes      | Field label                                          |
| `fields[].type`    | enum   | Yes      | Field type (see `addField` for full list)            |
| `fields[].options` | array  | No       | For SELECT/MULTI_SELECT: `[{ _id?, value, color? }]` |

### Response

```json
{
	"created": 4,
	"fields": [
		{ "_id": "abc123def", "name": "Email", "type": "TEXT", "order": 0 },
		{
			"_id": "ghi456jkl",
			"name": "Priority",
			"type": "SELECT",
			"order": 1,
			"options": [
				{ "_id": "opt1", "value": "High", "color": "#e74c3c", "order": 0 },
				{ "_id": "opt2", "value": "Medium", "color": "#f39c12", "order": 1 },
				{ "_id": "opt3", "value": "Low", "color": "#2ecc71", "order": 2 }
			]
		},
		{ "_id": "mno789pqr", "name": "Due Date", "type": "DEADLINE", "order": 2 },
		{ "_id": "stu012vwx", "name": "Assignee", "type": "ASSIGNEE", "order": 3 }
	]
}
```

### Example

```typescript
await app.callServerTool({
	name: 'privos.lists.batchAddFields',
	arguments: {
		listId: 'list_001',
		fields: [
			{ name: 'Email', type: 'TEXT' },
			{
				name: 'Priority',
				type: 'SELECT',
				options: [
					{ value: 'High', color: '#e74c3c' },
					{ value: 'Medium', color: '#f39c12' },
					{ value: 'Low', color: '#2ecc71' },
				],
			},
			{ name: 'Due Date', type: 'DEADLINE' },
			{ name: 'Assignee', type: 'ASSIGNEE' },
		],
	},
});
```

---

## `privos.lists.batchCreateItems`

Create multiple items in a list in one call. Items default to the first stage unless `stageId` is specified per item. Every item is checked before any is created, so one refused item creates none.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

| Arg                    | Type   | Required | Description                                                                   |
| ---------------------- | ------ | -------- | ----------------------------------------------------------------------------- |
| `listId`               | string | Yes      | Target list ID                                                                |
| `items`                | array  | Yes      | Array of items to create                                                      |
| `items[].title`        | string | Yes      | Item title/name                                                               |
| `items[].description`  | string | No       | Item description                                                              |
| `items[].stageId`      | string | No       | Target stage ID (defaults to first stage, or the parent's stage)              |
| `items[].parentId`     | string | No       | Create as a sub-item of this existing item (not one created in the same call) |
| `items[].customFields` | array  | No       | Array of `{ fieldId: string, value: any }`                                    |

### Response

```json
{
	"created": 3,
	"items": [
		{ "_id": "item_abc", "name": "John Doe", "key": "CRM-1" },
		{ "_id": "item_def", "name": "Jane Smith", "key": "CRM-2" },
		{ "_id": "item_ghi", "name": "Bob Wilson", "key": "CRM-3" }
	]
}
```

### Example

```typescript
await app.callServerTool({
	name: 'privos.lists.batchCreateItems',
	arguments: {
		listId: 'list_001',
		items: [
			{
				title: 'John Doe',
				customFields: [
					{ fieldId: 'field_email', value: 'john@example.com' },
					{ fieldId: 'field_priority', value: 'priority_hight' },
				],
			},
			{
				title: 'Jane Smith',
				description: 'VIP customer',
				stageId: 'stage_002',
				customFields: [{ fieldId: 'field_email', value: 'jane@example.com' }],
			},
			{ title: 'Bob Wilson' },
		],
	},
});
```

---

## `privos.lists.getTimeline`

Read a list's Gantt timeline: each item's start and end date, milestone flag and finish-to-start predecessors (an item's predecessors must finish before it starts). Only non-archived items the caller may see are returned. A predecessor in another list, or one the caller cannot see, is listed under `externalPredecessors` with its stored label only; its dates are never read.

|           |              |
| --------- | ------------ |
| **Scope** | `lists:read` |

### Arguments

Pass `savedViewId`, or `startFieldId` and `endFieldId`; not both.

| Arg                 | Type   | Required    | Description                                                                                                                                                           |
| ------------------- | ------ | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `listId`            | string | Yes         | List ID                                                                                                                                                               |
| `savedViewId`       | string | Conditional | A Gantt saved view; its start, end and predecessor fields are used                                                                                                    |
| `startFieldId`      | string | Conditional | Start field (`DATE`, `DATE_TIME` or `DEADLINE`)                                                                                                                       |
| `endFieldId`        | string | Conditional | End field (`DATE`, `DATE_TIME` or `DEADLINE`), not the start field                                                                                                    |
| `dependencyFieldId` | string | No          | Predecessors field (`DEPENDENCIES`)                                                                                                                                   |
| `utcOffset`         | number | No          | Hours from UTC of the person the agent acts for (e.g. `7`). Decides the calendar day of a `DATE_TIME` / `DEADLINE`. Defaults to the caller's profile offset, else `0` |

### Response

`DATE` values are `YYYY-MM-DD`; `DATE_TIME` and `DEADLINE` values are ISO instants; an empty date is `null`. `isMilestone` is true when the item has one date only, or both on the same calendar day (a `DATE` is its UTC day, a `DATE_TIME` / `DEADLINE` its day at `utcOffset`, as the Gantt view shows it).

```json
{
	"listId": "list_001",
	"startFieldId": "field_start",
	"endFieldId": "field_end",
	"dependencyFieldId": "field_deps",
	"utcOffset": 7,
	"items": [
		{
			"itemId": "item_002",
			"key": "TASK-2",
			"name": "Build",
			"stageId": "stage_001",
			"start": "2026-10-06",
			"end": "2026-10-09",
			"isMilestone": false,
			"predecessors": ["item_001"],
			"externalPredecessors": [{ "itemId": "item_x", "name": "Design sign-off" }]
		}
	]
}
```

---

## `privos.lists.shiftItems`

Move items earlier or later by a whole number of days. Each date an item has moves by `days`, exactly like dragging its Gantt bar: a `DATE` moves by calendar days, a `DATE_TIME` or `DEADLINE` moves by whole days and keeps its time of day (at a fixed `utcOffset`, so across a daylight-saving change the local time moves by an hour). Items without a date are skipped.

The call is a dry run unless `dryRun` is `false`. A dry run writes nothing and runs every per-item permission gate, so `failed` already lists the items the caller could not edit. An applied shift writes through the `items.patchFields` path: the same per-item gates (isolated lists, `additionalEditors`, cross-team ACL, read-only rooms, grant policy), Time Machine history, automations, webhooks and the live list reload. Refused items are reported in `failed` and left unchanged; the rest are written.

|           |               |
| --------- | ------------- |
| **Scope** | `lists:write` |

### Arguments

Field arguments (`savedViewId` or `startFieldId` + `endFieldId`, optional `dependencyFieldId`) and `utcOffset` are the same as for `getTimeline`; `utcOffset` decides which items start after `afterItemId` ends.

| Arg                    | Type     | Required | Description                                                                                                                                |
| ---------------------- | -------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `listId`               | string   | Yes      | List ID                                                                                                                                    |
| `selector`             | object   | Yes      | Exactly one of `afterItemId`, `itemIds`, `filter`                                                                                          |
| `selector.afterItemId` | string   | —        | Without `cascade`: items whose first day is after this item's last day. With `cascade`: this item's transitive successors. It never moves. |
| `selector.itemIds`     | string[] | —        | These items (at most 200). Unknown or invisible ids are returned in `notFound`.                                                            |
| `selector.filter`      | object   | —        | A `queryItems` filter; archived items excluded. Must match at most 200 items.                                                              |
| `days`                 | integer  | Yes      | Non-zero, between -3650 and 3650. Negative moves earlier.                                                                                  |
| `cascade`              | boolean  | No       | Also move every item that depends on a selected item, transitively. Needs a predecessors field. Default `false`.                           |
| `dryRun`               | boolean  | No       | Default `true`. Pass `false` to write.                                                                                                     |

A successor of X is an item whose predecessors field lists X. The traversal is cycle-safe and never follows a ref to another list or to an item the caller cannot see. A call that would move more than 200 items fails before any write.

### Response

```json
{
	"dryRun": true,
	"days": 3,
	"startFieldId": "field_start",
	"endFieldId": "field_end",
	"dependencyFieldId": "field_deps",
	"utcOffset": 7,
	"changes": [
		{
			"itemId": "item_002",
			"name": "Build",
			"from": { "start": "2026-10-06", "end": "2026-10-09" },
			"to": { "start": "2026-10-09", "end": "2026-10-12" }
		}
	],
	"failed": [{ "itemId": "item_003", "errorType": "error-not-allowed", "error": "You do not have permission to edit this item" }],
	"skipped": [{ "itemId": "item_004", "reason": "no start or end date" }],
	"notFound": [],
	"note": "Nothing was written. Show these changes to the user, then call again with dryRun: false to apply them."
}
```

With `dryRun: false`, `changes` lists the items written and `note` is absent. An apply is not idempotent: after an error or timeout, read `getTimeline` before retrying, or the items move twice.

### Example

```typescript
// "Shift every task after Design by 3 days": preview, then apply.
const args = { listId: 'list_001', savedViewId: 'view_gantt', selector: { afterItemId: 'item_001' }, days: 3, cascade: true };
await app.callServerTool({ name: 'privos.lists.shiftItems', arguments: args });
await app.callServerTool({ name: 'privos.lists.shiftItems', arguments: { ...args, dryRun: false } });
```
