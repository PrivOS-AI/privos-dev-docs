# Internal API Documentation

## Overview

The Internal API provides endpoints for server-to-server communication within the PrivOS Chat application. These APIs are designed for internal services, AI agents, and trusted integrations to manage core resources like Lists, Items, Stages, Documents, Rooms, and Users.

## Base URL
DEV:  

BASE_URL= https://privos-chat-dev.roxane.one/
API_KEY= dwVT6jcM-DI_Cs27nB4gaszG-wsBDvUJAkkuQt4RMuI

PROD:  

BASE_URL= https://privos.roxane.one/
API_KEY= ktRe4m91k8YZ2b5M90H_ZR8CzIouJvnsDgUMTw7n4UI


```
https://your-domain.com/api/v1/internal
```

## Authentication

### API Key Authentication

All Internal API endpoints require authentication via an API key passed in the request headers:

```http
x-api-key: <your-api-key>
```

**Note:** The header is `x-api-key` (not `Authorization: Bearer`). Both lowercase and uppercase `X-API-Key` are accepted.

**Example Requests:**

```bash
# Using curl
curl -X GET "https://your-domain.com/api/v1/internal/lists.list" \
  -H "x-api-key: YOUR_API_KEY"

# Using fetch
fetch('https://your-domain.com/api/v1/internal/lists.list', {
  headers: {
    'x-api-key': 'YOUR_API_KEY'
  }
})
```

### Getting an API Key

API keys are configured via environment variable:

**Environment Variable:** `PRIVOS_CHAT_INTERNAL_API_KEY`

Set this in your environment configuration:

```bash
# .env file
PRIVOS_CHAT_INTERNAL_API_KEY=your-secret-api-key-here

# Or export directly
export PRIVOS_CHAT_INTERNAL_API_KEY=your-secret-api-key-here
```

**Important:**
- API keys are currently only configurable via environment variable
- There is no UI-based API key generation
- Keep your API key secret and never expose it in client-side code
- Rotate API keys by updating the environment variable and restarting the server

### Authentication Flow

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Client    │────▶│   Request     │────▶│  Middleware  │
│             │     │ with         │     │  Validates   │
│   Server     │     │ x-api-key     │     │  API Key     │
└─────────────┘     └──────────────┘     └─────────────┘
                                                    │
                                                    ▼
                                              ┌─────────────────────┐
                                              │                     │
                                              │  Valid?             │
                                              │  ────────┬─────────│
                                              │         │         │
                                              │ Yes      No       │
                                              │  │       │         │
                                              │  ▼       ▼         │
                                              │ Pass    Return    │
                                              │        401       │
                                              └─────────────────────┘
```

### Error Responses

**Missing API Key:**

```json
{
  "success": false,
  "error": "error-unauthorized",
  "message": "Internal API key required"
}
```

**Invalid API Key:**

```json
{
  "success": false,
  "error": "error-unauthorized",
  "message": "Invalid Internal API key"
}
```

## Rate Limiting

Some endpoints have rate limiting applied:
- **Batch Operations**: 5 requests per minute per IP
- **Standard Operations**: 60 requests per minute per IP

Rate limit headers are included in responses:
```http
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1705880400
```

## Common Response Format

### Success Response

```json
{
  "success": true,
  "data": {
    // Response data specific to endpoint
  }
}
```

### Error Response

```json
{
  "success": false,
  "error": "error-code",
  "errorType": "error-type",
  "message": "Human readable error message"
}
```

### Common Error Codes

| Error Code | Description |
|------------|-------------|
| `error-invalid-params` | Required parameters are missing or invalid |
| `error-unauthorized` | Invalid or missing API key |
| `error-forbidden` | Insufficient permissions |
| `error-not-found` | Resource not found |
| `error-rate-limit-exceeded` | Rate limit exceeded |

## Pagination

List endpoints support pagination via query parameters:

```
GET /api/v1/internal/items.byListId?listId=xxx&offset=0&count=50
```

### Response

```json
{
  "success": true,
  "data": {
    "items": [...],
    "count": 50,
    "offset": 0,
    "total": 150
  }
}
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `offset` | number | 0 | Number of items to skip |
| `count` | number | 50 | Number of items to return (max: 100) |

## Conventions

### Resource Identifiers

Most resources support multiple identifier types:

| Identifier | Description | Example |
|------------|-------------|---------|
| `_id` | Internal MongoDB ID | `65a1b2c3d4e5f6g7h8i9j0k1` |
| `key` | Human-readable unique key | `project-alpha-backlog` |
| `templateKey` | Template-based identifier | `sprint-backlog` |

### Partial Updates

For endpoints supporting partial updates (like custom fields), only the fields specified in the request will be updated. Other fields remain unchanged.

### Batch Operations

Batch operations process items in chunks for performance. Response includes processing statistics:

```json
{
  "success": true,
  "data": {
    "message": "Batch operation completed",
    "totalProcessed": 100,
    "totalFailed": 2,
    "errors": [...],
    "processingTime": 2340
  }
}
```

## API Endpoints

| Resource | Documentation | Description |
|----------|---------------|-------------|
| **Lists** | [lists.md](./lists.md) | Manage Kanban-style lists with custom fields |
| **Items** | [items.md](./items.md) | CRUD operations on list items, including batch operations |
| **Stages** | [stages.md](./stages.md) | Manage workflow stages within lists |
| **Documents** | [documents.md](./documents.md) | Document management with versioning |
| **Rooms** | [rooms.md](./rooms.md) | Room queries and member information |
| **Users** | [users.md](./users.md) | User listing with positions and skills |
| **Channels** | [channels.md](./channels.md) | Channel member listing |
| **Groups** | [groups.md](./groups.md) | Private group member listing |
| **AI Messages** | [ai-messages.md](./ai-messages.md) | Client-facing AI chat messaging and context-aware session resume |
| **Agent Chat** | [agent-chat.md](./agent-chat.md) | Send messages as bot users |
| **Agent Chat Session** | [agent-chat-session.md](./agent-chat-session.md) | Canvas artifacts and context-aware sessions in AI conversations |
| **Shared Folders** | [shared-folders.md](./shared-folders.md) | Cross-room folder sharing with permission control |
| **File Backups** | [file-backups.md](./file-backups.md) | Time Machine file backup, restore, and admin config |

## Versioning

The API is versioned via the URL path: `/api/v1/internal/`.

## Support

For issues or questions:
- GitHub Issues: https://github.com/PrivOS-AI/privos-chat/issues
- Documentation: https://docs.privos.ai
