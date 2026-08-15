# MCP App Platform — API Documentation

Detailed API reference organized by category.

## REST Endpoints

| Document | Description |
|----------|-------------|
| [App Management](./rest-app-management.md) | Connect, list, get, refresh, delete apps |
| [Credential Push (outbound)](./rest-credential-push.md) | Hub → App `/.well-known/mcp/register` push contract |
| [Relay Apps](./rest-relay-apps.md) | Pairing, relay status, OAuth token |
| [Installations](./rest-installations.md) | Install/uninstall apps in rooms |
| [Settings](./rest-settings.md) | Update app settings and permissions |
| [Tool Execution](./rest-tool-call.md) | Execute MCP tools via REST |
| [UI Resources](./rest-ui-resource.md) | Fetch app UI HTML resources |
| [Item Query](./items-query.md) | `items.query` — filtered, cursor-paginated item retrieval |

## PrivOS MCP Tools

REST-first is the primary integration path: data operations (lists, items, files,
folders, messages, rooms, users, stages) are reached through `/api/v1` via the granted
OAuth scopes — see [Auth & REST Integration](../auth-and-rest-integration.md) and the
REST endpoints above. The MCP tools below remain for capabilities with **no REST
equivalent**:

| Document | Description |
|----------|-------------|
| [Context Tools](./tools-context.md) | Get current user/room context |
| [App Storage Tools](./tools-app.md) | `mcpapp.app.*` — per-app `localData` key/value store |
| [Bot Tools](./tools-bot.md) | Use the manifest-declared, admin-created installation-owned agent bot in approved Rooms, or legacy token-selected messaging |
| [Database Tools](./tools-database.md) | `mcpapp.db.*` — schema, CRUD, query, aggregate |

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
| `lists:read` | Read lists, items, stages, and field definitions |
| `lists:query` | Run filtered, paginated queries over list items (`POST items.query`). Separate from `lists:read` because it is a POST, and every `*:read` scope is GET-only |
| `lists:write` | Create/update/delete lists, items, stages, and fields |
| `files:read` | Read files and folders |
| `files:write` | Upload/update/delete files and folders |
| `messages:read` | Read messages in authorized rooms |
| `messages:send` | Send messages in authorized rooms |
| `users:read` | Read user profiles |
| `rooms:read` | Read room metadata and members |
| `rooms:write` | Create/update rooms |
| `bot:room:join` | Add the installation-owned bot (declared in the manifest `agentBot` block, created by a workspace admin) to the exact approved Room (interactive user only) |
| `bot:identity:read` | Read safe identity only when that bot has ordinary membership in the exact approved Room |
| `bot:message:send` | Send messages/DMs/attachments as a bot (runtime identity via bot token) |
| `db:schema:read` | List collections and read schema definitions |
| `db:schema:write` | Register/update/drop app DB collections and schemas |
| `db:read` | Read records, query, count, aggregate |
| `db:write` | Create/update/delete records (soft-delete) |
| `sandbox:ai-chat` | List a room's AI Chat sessions and read their message history |
| `sandbox:ai-chat:write` | Send messages in a room's AI Chat, start the agent generation, and cancel an in-flight run |
| `sandbox:agent-sets:upload` | Submit agent-set archives to the workspace Agent Factory (`agents.sandbox.agentSets.preview` / `.confirm`); endpoints additionally require the workspace-admin `manage-privos-agent-sets` permission |

### Scope-free tools

A few tools bypass the OAuth scope check — they only require the app to be installed and the user to have room access:

- `mcpapp.context.get` — reads current user/room context
- `mcpapp.bot.getMe` — validates a bot token (runtime identity check)
- `mcpapp.app.*` — per-app `localData` store (scoped to the app itself)
