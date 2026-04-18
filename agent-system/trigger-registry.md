# Trigger Registry

Unified automation system for agents. Three trigger types share a single execution path: inject a synthetic message into the agent room, where the existing reply handler processes it.

## Trigger Types

| Type | Source | Example |
|------|--------|---------|
| `cron` | Heartbeat cron (every 60s) | "Every hour, scan trending news" |
| `webhook` | External HTTP POST | Telegram message, email received, GitHub push |
| `event` | Internal RC events | New message in room X, user joined, file uploaded |

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   TRIGGER SOURCES                    │
├──────────────┬──────────────────┬───────────────────┤
│  Cron        │  Webhook         │  RC Event         │
│  (heartbeat  │  (POST /v1/      │  (afterSave       │
│   every 60s) │   agents.webhook │   Message, etc.)  │
│              │   /:token)       │                   │
└──────┬───────┴────────┬─────────┴─────────┬─────────┘
       │                │                   │
       ▼                ▼                   ▼
┌─────────────────────────────────────────────────────┐
│         injectTriggerMessage(agent, trigger, ctx)    │
│  → sendMessage() with t:'agent-trigger' into room    │
└──────────────────────────┬──────────────────────────┘
                           ▼
              agent-room-reply-handler (existing)
                           ▼
                    Privos Brain → response
```

## Trigger Schema

Triggers are stored as an array in `customFields.agentTriggers[]` on the bot user document. Max 5 triggers per agent.

```ts
interface IAgentTrigger {
  id: string;                    // Random.id()
  type: 'cron' | 'webhook' | 'event';
  enabled: boolean;
  lastRunAt?: Date;

  // type=cron
  schedule?: string;             // key from INTERVALS map
  prompt?: string;               // static prompt injected (max 500 chars)

  // type=webhook
  webhookToken?: string;         // unique token for inbound URL
  webhookSecret?: string;        // HMAC signature verification secret
  prompt?: string;               // prompt sent with webhook payload as context

  // type=event
  event?: BotWebhookEvent;       // 'message.new', 'file.created', etc.
  sourceRoomId?: string;         // filter to specific room (empty = all rooms)
  promptTemplate?: string;       // template sent with event data as context
}
```

## Shared Execution: injectTriggerMessage()

File: `server/services/agent-trigger-injector.ts`

Single funnel for all trigger types. Sends a synthetic message into the agent room.

```
injectTriggerMessage({ agentId, triggerId, prompt, context? })
        │
        ├─ Resolve room: agent-room-{agentId}
        ├─ Resolve bot user
        ├─ Compose: prompt + "\n\n---\n" + context (if provided)
        │
        ▼
sendMessage(botUser, {
  rid: roomId,
  msg: composed,
  t: 'agent-trigger',    // custom message type
  triggerId,
  groupable: false
})
```

The `t: 'agent-trigger'` marker allows the reply handler to distinguish trigger messages from regular bot messages and process them instead of skipping.

---

## Cron Triggers

File: `server/cron/agentHeartbeat.ts`

### How It Works

A single cron job (`agent-heartbeat`) runs every 60 seconds via `@rocket.chat/cron` (Agenda). Each tick:

1. Query all bot users with enabled cron triggers
2. For each trigger, check if enough time has elapsed since `lastRunAt`
3. Atomically update `lastRunAt` (prevents double-fire across server instances)
4. If update succeeded → `injectTriggerMessage()`

### Predefined Intervals

| Key | Interval |
|-----|----------|
| `every_5m` | 5 minutes (300,000 ms) |
| `every_15m` | 15 minutes (900,000 ms) |
| `every_30m` | 30 minutes (1,800,000 ms) |
| `every_1h` | 1 hour (3,600,000 ms) |
| `every_6h` | 6 hours (21,600,000 ms) |
| `every_12h` | 12 hours (43,200,000 ms) |
| `every_24h` | 24 hours (86,400,000 ms) |

No raw cron expressions allowed — safer UX, simpler validation.

### Double-Fire Prevention

Uses atomic MongoDB update with optimistic locking:

```js
Users.col.updateOne(
  {
    '_id': agent._id,
    'customFields.agentTriggers': {
      $elemMatch: {
        id: trigger.id,
        $or: [
          { lastRunAt: trigger.lastRunAt },       // match current value
          { lastRunAt: { $exists: false } }       // or never run
        ]
      }
    }
  },
  { $set: { 'customFields.agentTriggers.$.lastRunAt': new Date() } }
)
```

If `modifiedCount === 0`, another instance already fired it → skip.

### MongoDB Query (per tick)

```js
Users.col.find({
  'type': 'bot',
  'customFields.isAgentBot': true,
  'customFields.agentTriggers': {
    $elemMatch: { type: 'cron', enabled: true }
  }
})
```

---

## Webhook Triggers

File: `app/api/server/v1/agent-trigger-endpoints.ts`

### How It Works

Each webhook trigger gets a unique `webhookToken` and `webhookSecret` on creation. External services POST to:

```
POST /api/v1/agents.webhook/{webhookToken}
```

No authentication required (token-based). The request body is passed as `context` to the trigger prompt.

### HMAC Signature Verification

Optional. If `webhookSecret` exists on the trigger:

```
Header: x-webhook-signature: <hex>
Expected: HMAC-SHA256(webhookSecret, JSON.stringify(body))
```

If signature doesn't match → 401 Unauthorized.

### Rate Limiting

In-memory rate limiter: **60 requests per minute per agent**.

```js
Map<agentId, { count: number; resetAt: number }>
```

Resets every 60 seconds. Returns `error-too-many-requests` when exceeded.

### Webhook URL Format

```
{ROOT_URL}/api/v1/agents.webhook/{webhookToken}
```

Displayed in Agent Settings tab with copy-to-clipboard button. Also returned in `agents.triggers.add` response for webhook type.

---

## Event Triggers

File: `server/services/agent-event-trigger-handler.ts`

### How It Works

RC events (currently `message.new` via `afterSaveMessage`) are dispatched to agents with matching event triggers.

```
afterSaveMessage callback
        │
        ├─ Skip system messages (message.t exists)
        ├─ Skip bot user messages
        │
        ▼
dispatchAgentEventTriggers({ event: 'message.new', roomId, data })
        │
        ├─ Query agents with matching event triggers
        ├─ For each matching trigger:
        │   ├─ Check sourceRoomId filter (skip if doesn't match)
        │   ├─ Self-loop prevention (skip if roomId = agent-room-{agentId})
        │   ├─ Cooldown check (30s per trigger)
        │   ├─ Update lastRunAt
        │   └─ injectTriggerMessage() with event data as context
```

### Supported Events

All `BotWebhookEvent` types are valid for event triggers:

| Category | Events |
|----------|--------|
| Message | `message.new`, `message.edited`, `message.deleted`, `message.mention`, `message.bot_mention` |
| Room | `room.joined`, `room.left` |
| User | `user.joined`, `user.left` |
| List Item | `list.item.created`, `list.item.deleted`, `list.item.stage_changed`, `list.item.attributes_changed` |
| File | `file.created`, `file.updated`, `file.deleted` |
| Folder | `folder.created`, `folder.deleted`, `folder.renamed` |

Currently only `message.new` is dispatched via `afterSaveMessage`. Additional events can be wired by adding dispatch calls to their respective callbacks.

### Self-Loop Prevention

Events from the agent's own room (`agent-room-{agentId}`) are always skipped. This prevents an agent's own replies from triggering its event triggers.

### Cooldown

In-memory cooldown per trigger: **30 seconds minimum between fires**.

```js
Map<`${agentId}:${triggerId}`, timestamp>
```

Prevents rapid-fire when many events happen in quick succession.

### Context Payload

Event data is serialized as context appended to the trigger's `promptTemplate`:

```
Event: message.new
Room: GENERAL
{
  "message": { "id": "...", "msg": "...", "userId": "...", "username": "..." },
  "room": { "id": "GENERAL", "name": "general", "type": "c" }
}
```

---

## Startup Registration

File: `server/configuration/bot.ts`

All trigger subsystems are registered during `configureBotWebhooks()`:

1. `initializeBotWebhookTriggers()` — legacy bot webhook queue
2. `afterSaveMessage` callback → `handleAgentRoomMessage()` (agent auto-reply)
3. `afterSaveMessage` callback → `dispatchAgentEventTriggers()` (event triggers)
4. `agentHeartbeatCron()` — heartbeat cron job

Console output on successful startup:
```
[BOT-WEBHOOKS] Bot webhook system initialized
[AGENT-REPLY] Agent room auto-reply registered
[AGENT-EVENT-TRIGGER] Agent event triggers registered
[AGENT-HEARTBEAT] Agent heartbeat cron registered
```
