# Bot Webhooks Documentation

Complete reference for webhook events and payload structures in Privos Chat.

## Quick Links

- [Event Types Reference](./supported-webhook-events-reference.md) - Complete list of all 18 event types
- [Payload Examples](./webhook-payload-examples.md) - Detailed JSON examples for each event

## Overview

Privos Chat webhooks enable real-time event notifications to external services. The system supports **18 event types** across 6 categories:

- **Message Events** (4): `message.new`, `message.edited`, `message.deleted`, `message.mention`
- **Room Events** (2): `room.joined`, `room.left`
- **User Events** (2): `user.joined`, `user.left`
- **List Item Events** (4): `list.item.*` (create, delete, stage_changed, attributes_changed)
- **File Events** (3): `file.created`, `file.updated`, `file.deleted`
- **Folder Events** (3): `folder.created`, `folder.deleted`, `folder.renamed`

## Key Features

- **Fire-and-forget delivery**: Webhooks don't block API responses
- **Reliable retry logic**: Exponential backoff with 24 attempts
- **Redis queue persistence**: Survives server restarts
- **Per-bot isolation**: Independent processing for each bot
- **Room-based filtering**: Bots only receive events from rooms they're in
- **HMAC signatures**: All payloads signed with SHA256

## Webhook Setup

Configure webhooks via the Bot API using `/api/v1/bot/setWebhook`:

```json
{
  "url": "https://your-server.com/webhook",
  "events": ["message.new", "file.created", "folder.created"],
  "secret": "your-webhook-secret"
}
```

## Important Notes

**Security**:
- File events use `folderPath` (display path) instead of internal MinIO paths
- Both `stage_changed` and `attributes_changed` can fire for the same list item update
- `changedFieldIds` only present for custom field changes in `attributes_changed`

**Cascade Operations**:
- Folder cascade delete emits individual `file.deleted` events per contained file, plus one `folder.deleted`

**Implementation Details**:
- All events include `actor` field (user who triggered it)
- `stage` field optional in `list.item.deleted` when item's stage was deleted
- `previousStage` optional in `list.item.stage_changed`
