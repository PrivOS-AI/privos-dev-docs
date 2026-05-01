# MCP App Platform — API Documentation

Detailed API reference organized by category.

## REST Endpoints

| Document | Description |
|----------|-------------|
| [App Management](./rest-app-management.md) | Connect, list, get, refresh, delete apps |
| [Relay Apps](./rest-relay-apps.md) | Pairing, relay status, OAuth token |
| [Installations](./rest-installations.md) | Install/uninstall apps in rooms |
| [Settings](./rest-settings.md) | Update app settings and permissions |
| [Tool Execution](./rest-tool-call.md) | Execute MCP tools via REST |
| [UI Resources](./rest-ui-resource.md) | Fetch app UI HTML resources |

## PrivOS MCP Tools

| Document | Description |
|----------|-------------|
| [Context Tools](./tools-context.md) | Get current user/room context |
| [Lists & Items Tools](./tools-lists.md) | CRUD for lists, items, custom fields |
| [Stages Tools](./tools-stages.md) | Kanban stages management |
| [Files Tools](./tools-files.md) | File operations |
| [Folders Tools](./tools-folders.md) | Folder operations |
| [Messages Tools](./tools-messages.md) | Read and send messages |
| [Rooms Tools](./tools-rooms.md) | Room metadata and members |
| [Users Tools](./tools-users.md) | User profile information |

## Authentication

All REST endpoints require authentication via `X-Auth-Token` and `X-User-Id` headers.

Admin endpoints additionally require the `manage-oauth-apps` permission.

## Base URL

```
https://{your-privos-host}/api/v1/
```

## Common Response Format

All endpoints return JSON. Error responses follow this format:

```json
{
  "success": false,
  "error": "Error message description"
}
```

Successful responses include `"success": true` along with the endpoint-specific data.

## Scope Taxonomy

| Scope | Grants |
|-------|--------|
| `lists:read` | Read lists, items, stages, and fields |
| `lists:write` | Create/update/delete lists, items, stages, and fields |
| `files:read` | Read files and folders |
| `files:write` | Upload/update/delete files and folders |
| `messages:read` | Read messages in authorized rooms |
| `messages:send` | Send messages in authorized rooms |
| `users:read` | Read user profiles |
| `rooms:read` | Read room metadata and members |
| `rooms:write` | Create/update rooms |
