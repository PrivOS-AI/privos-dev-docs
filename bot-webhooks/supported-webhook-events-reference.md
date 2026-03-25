# Supported Webhook Events Reference

Complete reference of all 18 webhook event types organized by category.

## Message Events (4)

| Event | Description | Triggered When |
|-------|-------------|----------------|
| `message.new` | New message posted | A user sends a message in a room with the bot |
| `message.edited` | Message edited | A user edits their message in a room with the bot |
| `message.deleted` | Message deleted | A user deletes their message in a room with the bot |
| `message.mention` | Message with mention | A user sends a message with @mentions in a room with the bot |

## Room Events (2)

| Event | Description | Triggered When |
|-------|-------------|----------------|
| `room.joined` | Bot joined room | The bot itself is added to a room |
| `room.left` | Bot left room | The bot itself is removed from a room |

## User Events (2)

| Event | Description | Triggered When |
|-------|-------------|----------------|
| `user.joined` | User joined room | A user (not bot) joins a room where the bot is present |
| `user.left` | User left room | A user (not bot) leaves a room where the bot is present |

## List Item Events (4)

| Event | Description | Triggered When |
|-------|-------------|----------------|
| `list.item.created` | List item created | An item is created in a list |
| `list.item.deleted` | List item deleted | An item is deleted from a list |
| `list.item.stage_changed` | List item stage changed | An item is moved to a different stage |
| `list.item.attributes_changed` | List item attributes changed | An item's name, description, or custom fields are modified |

## File Events (3)

| Event | Description | Triggered When |
|-------|-------------|----------------|
| `file.created` | File uploaded | A file is uploaded to a room |
| `file.updated` | File renamed/moved | A file is renamed or moved to a different folder |
| `file.deleted` | File deleted | A file is deleted from a room |

## Folder Events (3)

| Event | Description | Triggered When |
|-------|-------------|----------------|
| `folder.created` | Folder created | A new folder is created in a room |
| `folder.deleted` | Folder deleted | A folder is deleted (with cascading file deletions) |
| `folder.renamed` | Folder renamed | A folder is renamed |

## Important Notes

**Bot Message Exclusion**: Bot messages are excluded from triggering webhooks to prevent infinite loops. Bots only receive events from regular users.

**Actor Field**: List, file, and folder events include an `actor` field showing which user triggered the event.

**File Paths**: File events use `folderPath` (display path) instead of internal MinIO paths for security reasons.

**Mention Event Structure**: The `message.mention` event uses a different payload structure with a `form` object instead of `room`, and raw field names (`_id`, `rid`, `msg`).

**Inline Buttons**: Buttons with `bot-event` action trigger `message.mention` webhooks with `metadata` in the `form` object.
