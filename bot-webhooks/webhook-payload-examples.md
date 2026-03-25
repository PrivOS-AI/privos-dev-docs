# Webhook Payload Examples

Detailed payload examples for all webhook event types.

## Message Events

### New Message (`message.new`)

```json
{
    "event": "message.new",
    "timestamp": "2024-01-05T11:30:00.000Z",
    "bot": {
        "id": "694b67f394ee87bd045c2ad1",
        "username": "Bot11",
        "name": "My Bot"
    },
    "room": {
        "id": "GENERAL",
        "name": "general",
        "type": "c"
    },
    "message": {
        "id": "msg123",
        "text": "Hello Bot11!",
        "userId": "user123",
        "username": "john",
        "createdAt": "2024-01-05T11:30:00.000Z"
    }
}
```

### Message Edited (`message.edited`)

```json
{
    "event": "message.edited",
    "timestamp": "2024-01-05T11:31:00.000Z",
    "bot": { ... },
    "room": { ... },
    "message": {
        "id": "msg123",
        "text": "Hello Bot11! (edited)",
        "userId": "user123",
        "username": "john",
        "createdAt": "2024-01-05T11:30:00.000Z"
    }
}
```

### Message Mention (`message.mention`)

```json
{
    "event": "message.mention",
    "timestamp": "2024-01-05T11:30:00.000Z",
    "bot": { ... },
    "form": {
        "roomId": "GENERAL",
        "roomName": "general",
        "roomSessionKey": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
        "privosEndpointUrl": "https://your-privos-chat.com"
    },
    "message": {
        "_id": "msg123",
        "rid": "GENERAL",
        "msg": "Hello @john @jane",
        "mentions": [
            { "_id": "user123", "username": "john" },
            { "_id": "user456", "username": "jane" }
        ]
    }
}
```

**Note**: The `message.mention` event uses different payload structure than other message events. It includes `form` object instead of `room`, and message uses raw field names (`_id`, `rid`, `msg`).

### Inline Button Click (`message.mention` with metadata)

When an inline button with `bot-event` action is clicked:

```json
{
    "event": "message.mention",
    "timestamp": "2024-01-05T11:35:00.000Z",
    "bot": { ... },
    "form": {
        "roomId": "GENERAL",
        "roomName": "general",
        "roomSessionKey": "...",
        "privosEndpointUrl": "https://your-privos-chat.com",
        "metadata": {
            "action": "approve",
            "requestId": "req_001",
            "timestamp": "2024-01-05T11:35:00.000Z"
        }
    },
    "message": { ... }
}
```

## List Item Events

### List Item Created (`list.item.created`)

```json
{
    "event": "list.item.created",
    "timestamp": "2024-01-05T12:00:00.000Z",
    "bot": { ... },
    "room": { ... },
    "item": {
        "id": "item123",
        "name": "Task title",
        "description": "Task description"
    },
    "list": {
        "id": "list456",
        "name": "My List"
    },
    "stage": {
        "id": "stage789",
        "name": "To Do"
    },
    "actor": {
        "id": "user123",
        "username": "john",
        "name": "John Doe"
    }
}
```

### List Item Stage Changed (`list.item.stage_changed`)

```json
{
    "event": "list.item.stage_changed",
    "timestamp": "2024-01-05T12:05:00.000Z",
    "bot": { ... },
    "room": { ... },
    "item": { ... },
    "list": { ... },
    "newStage": {
        "id": "stage790",
        "name": "In Progress"
    },
    "previousStage": {
        "id": "stage789",
        "name": "To Do"
    },
    "actor": { ... }
}
```

### List Item Attributes Changed (`list.item.attributes_changed`)

```json
{
    "event": "list.item.attributes_changed",
    "timestamp": "2024-01-05T12:10:00.000Z",
    "bot": { ... },
    "room": { ... },
    "item": { ... },
    "list": { ... },
    "stage": { ... },
    "changedFields": {
        "name": { "previous": "Old title", "current": "New title" },
        "customField1": { "previous": "value1", "current": "value2" }
    },
    "actor": { ... }
}
```

## File Events

### File Created (`file.created`)

```json
{
    "event": "file.created",
    "timestamp": "2024-01-05T12:15:00.000Z",
    "bot": { ... },
    "room": { ... },
    "file": {
        "id": "file123",
        "name": "document.pdf",
        "size": 2048576,
        "type": "application/pdf",
        "channelId": "channel456",
        "folderId": "folder789",
        "folderPath": "/My Folder/Subfolder"
    },
    "actor": { ... }
}
```

### File Updated (`file.updated`)

```json
{
    "event": "file.updated",
    "timestamp": "2024-01-05T12:20:00.000Z",
    "bot": { ... },
    "room": { ... },
    "file": {
        "id": "file123",
        "name": "document-renamed.pdf",
        "size": 2048576,
        "type": "application/pdf",
        "channelId": "channel456",
        "folderId": "folder790",
        "folderPath": "/My Folder/Another Subfolder"
    },
    "changes": {
        "previousName": "document.pdf",
        "newName": "document-renamed.pdf",
        "previousFolderId": "folder789",
        "newFolderId": "folder790"
    },
    "actor": { ... }
}
```

### File Deleted (`file.deleted`)

```json
{
    "event": "file.deleted",
    "timestamp": "2024-01-05T12:25:00.000Z",
    "bot": { ... },
    "room": { ... },
    "file": {
        "id": "file123",
        "name": "document.pdf",
        "size": 2048576,
        "type": "application/pdf",
        "channelId": "channel456",
        "folderId": "folder789",
        "folderPath": "/My Folder/Subfolder"
    },
    "actor": { ... }
}
```

## Folder Events

### Folder Created (`folder.created`)

```json
{
    "event": "folder.created",
    "timestamp": "2024-01-05T12:30:00.000Z",
    "bot": { ... },
    "room": { ... },
    "folder": {
        "id": "folder123",
        "name": "My Documents",
        "channelId": "channel456",
        "parentFolderId": "folder456",
        "folderPath": "/Parent Folder/My Documents"
    },
    "actor": { ... }
}
```

### Folder Deleted (`folder.deleted`)

```json
{
    "event": "folder.deleted",
    "timestamp": "2024-01-05T12:35:00.000Z",
    "bot": { ... },
    "room": { ... },
    "folder": {
        "id": "folder123",
        "name": "My Documents",
        "channelId": "channel456",
        "parentFolderId": "folder456",
        "folderPath": "/Parent Folder/My Documents"
    },
    "actor": { ... }
}
```

### Folder Renamed (`folder.renamed`)

```json
{
    "event": "folder.renamed",
    "timestamp": "2024-01-05T12:40:00.000Z",
    "bot": { ... },
    "room": { ... },
    "folder": {
        "id": "folder123",
        "name": "My Documents - Updated",
        "channelId": "channel456",
        "parentFolderId": "folder456",
        "folderPath": "/Parent Folder/My Documents - Updated"
    },
    "previousName": "My Documents",
    "actor": { ... }
}
```
