# Trigger API Reference

REST endpoints for managing agent triggers and receiving webhooks.

All endpoints except `agents.webhook/:token` require authentication via `X-Auth-Token` header.

## Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/v1/agents.triggers.list` | Yes | List triggers for an agent |
| POST | `/v1/agents.triggers.add` | Yes | Add a new trigger |
| POST | `/v1/agents.triggers.update` | Yes | Update an existing trigger |
| POST | `/v1/agents.triggers.remove` | Yes | Remove a trigger |
| POST | `/v1/agents.triggers.run` | Yes | Manually fire a trigger |
| POST | `/v1/agents.webhook/:token` | No | Receive external webhook |

## Authorization

All authenticated endpoints verify **bot ownership**: the requesting user must be the bot's `_createdBy` user OR have `admin` permission.

---

## GET /v1/agents.triggers.list

List all triggers for an agent. The `webhookSecret` field is stripped from responses for security.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `botId` | string | Yes | Agent bot user ID |

**Response:**

```json
{
  "triggers": [
    {
      "id": "abc123",
      "type": "cron",
      "enabled": true,
      "schedule": "every_1h",
      "prompt": "Scan trending news",
      "lastRunAt": "2026-04-16T10:00:00.000Z"
    },
    {
      "id": "def456",
      "type": "webhook",
      "enabled": true,
      "prompt": "Process incoming data",
      "webhookToken": "xyz789",
      "lastRunAt": null
    },
    {
      "id": "ghi012",
      "type": "event",
      "enabled": true,
      "event": "message.new",
      "sourceRoomId": "GENERAL",
      "promptTemplate": "Triage this message",
      "lastRunAt": null
    }
  ],
  "success": true
}
```

---

## POST /v1/agents.triggers.add

Add a new trigger. Maximum 5 triggers per agent.

**Body Parameters (common):**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `botId` | string | Yes | Agent bot user ID |
| `type` | string | Yes | `cron`, `webhook`, or `event` |

**Body Parameters (type=cron):**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `schedule` | string | Yes | Interval key (see below) |
| `prompt` | string | Yes | What the agent should do (max 500 chars) |

**Body Parameters (type=webhook):**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | Yes | What the agent should do with incoming data (max 500 chars) |

`webhookToken` and `webhookSecret` are auto-generated on creation.

**Body Parameters (type=event):**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `event` | string | Yes | Event type (see supported events) |
| `sourceRoomId` | string | No | Filter to specific room (empty = all rooms) |
| `promptTemplate` | string | Yes | What the agent should do (max 500 chars) |

**Valid Schedules:**

`every_5m`, `every_15m`, `every_30m`, `every_1h`, `every_6h`, `every_12h`, `every_24h`

**Valid Events:**

`message.new`, `message.edited`, `message.deleted`, `message.mention`, `message.bot_mention`, `room.joined`, `room.left`, `user.joined`, `user.left`, `list.item.created`, `list.item.deleted`, `list.item.stage_changed`, `list.item.attributes_changed`, `file.created`, `file.updated`, `file.deleted`, `folder.created`, `folder.deleted`, `folder.renamed`

**Response (cron/event):**

```json
{
  "success": true,
  "trigger": {
    "id": "abc123",
    "type": "cron",
    "enabled": true,
    "schedule": "every_1h",
    "prompt": "Scan trending news"
  }
}
```

**Response (webhook):**

```json
{
  "success": true,
  "trigger": {
    "id": "def456",
    "type": "webhook",
    "enabled": true,
    "prompt": "Process incoming data",
    "webhookToken": "xyz789",
    "webhookSecret": "secret..."
  },
  "webhookUrl": "https://chat.example.com/api/v1/agents.webhook/xyz789"
}
```

**Errors:**

| Error | Cause |
|-------|-------|
| `botId and type are required` | Missing required fields |
| `type must be cron, webhook, or event` | Invalid type |
| `Bot not found or not authorized` | Bot doesn't exist or user doesn't own it |
| `Maximum 5 triggers per agent` | Limit reached |
| `Invalid schedule` | Schedule key not in INTERVALS map |
| `prompt is required for cron triggers` | Empty prompt |
| `prompt must be under 500 characters` | Prompt too long |
| `Invalid event` | Event not in valid events list |
| `promptTemplate is required for event triggers` | Empty prompt template |

---

## POST /v1/agents.triggers.update

Update fields on an existing trigger.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `botId` | string | Yes | Agent bot user ID |
| `triggerId` | string | Yes | Trigger ID |
| `schedule` | string | No | New schedule (cron only) |
| `prompt` | string | No | New prompt (max 500 chars) |
| `promptTemplate` | string | No | New prompt template (max 500 chars) |
| `enabled` | boolean | No | Enable/disable toggle |
| `event` | string | No | New event type (event only) |
| `sourceRoomId` | string | No | New source room filter |

Only provided fields are updated. At least one updatable field is required.

**Response:**

```json
{
  "success": true
}
```

---

## POST /v1/agents.triggers.remove

Remove a trigger from an agent.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `botId` | string | Yes | Agent bot user ID |
| `triggerId` | string | Yes | Trigger ID to remove |

**Response:**

```json
{
  "success": true
}
```

---

## POST /v1/agents.triggers.run

Manually fire a trigger immediately. Updates `lastRunAt`.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `botId` | string | Yes | Agent bot user ID |
| `triggerId` | string | Yes | Trigger ID to fire |

The trigger's prompt (or promptTemplate) is injected into the agent room with context `"Manual trigger run"`.

**Response:**

```json
{
  "success": true
}
```

**Errors:**

| Error | Cause |
|-------|-------|
| `Trigger not found` | Trigger ID doesn't exist on this bot |
| `Trigger has no prompt` | Trigger has neither prompt nor promptTemplate |
| `Failed to inject trigger message` | Agent room or bot user not found |

---

## POST /v1/agents.webhook/:token

Public endpoint for receiving external webhooks. No authentication — the unique token in the URL acts as the credential.

**URL Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `token` | string | Webhook trigger token (auto-generated) |

**Headers:**

| Header | Description |
|--------|-------------|
| `x-webhook-secret` | Shared secret (constant-time compare). |
| `Authorization: Bearer <secret>` | Alternate form, server accepts either. |

**Secret rules:**

- `emit_event` triggers — secret **mandatory**.
- `agentic_response` triggers — secret **optional** (only enforced when sent).

**Body:** Any JSON payload. The entire body is serialized and passed as context to the agent.

**Rate Limiting:**

60 requests per minute per agent (in-memory, resets every 60 seconds).

**Response (success):**

```json
{
  "success": true
}
```

**Response (errors):**

| Status | Cause |
|--------|-------|
| 404 | Token not found or trigger disabled |
| 401 | Secret missing or wrong |
| 429 | `error-too-many-requests` — rate limit exceeded |

**Example — agentic_response (no secret needed):**

```bash
curl -X POST https://chat.example.com/api/v1/agents.webhook/xyz789 \
  -H "Content-Type: application/json" \
  -d '{"from": "github", "event": "push", "repo": "my-app"}'
```

**Example — emit_event (secret required):**

```bash
curl -X POST https://chat.example.com/api/v1/agents.webhook/xyz789 \
  -H "Content-Type: application/json" \
  -H "x-webhook-secret: your-webhook-secret" \
  -d '{"from":"github","event":"push"}'
```
