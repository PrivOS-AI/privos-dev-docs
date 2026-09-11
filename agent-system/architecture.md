# Agent System — Architecture

## System Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        CLIENT (React)                            │
├──────────────┬──────────────────┬────────────────────────────────┤
│ Agent Builder│  Agent Room Chat │  Agent Settings Tab            │
│ (Modal)      │  (Messages)      │  (Trigger CRUD UI)             │
└──────┬───────┴────────┬─────────┴──────────┬─────────────────────┘
       │                │                    │
       ▼                ▼                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                      REST API (Express)                          │
├──────────────┬──────────────────┬────────────────────────────────┤
│ agents.*     │ afterSaveMessage │ agents.triggers.*              │
│ (builder,    │ callback         │ (list/add/update/remove/run)   │
│  create)     │                  │ agents.webhook/:token          │
└──────┬───────┴────────┬─────────┴──────────┬─────────────────────┘
       │                │                    │
       ▼                ▼                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                    SERVER SERVICES                                │
├──────────────┬──────────────────┬────────────────────────────────┤
│ PrivOS Sandbox │ Agent Room Reply │ Trigger System                 │
│ HTTP Client  │ Handler          │ ┌──────────────────────────┐   │
│ (sync +      │ (afterSaveMsg    │ │ injectTriggerMessage()   │   │
│  stream)     │  → Sandbox → resp) │ │ (shared funnel)          │   │
│              │                  │ ├──────────────────────────┤   │
│              │                  │ │ agentHeartbeatCron       │   │
│              │                  │ │ (every 60s, cron type)   │   │
│              │                  │ ├──────────────────────────┤   │
│              │                  │ │ dispatchAgentEventTriggers│  │
│              │                  │ │ (RC events, event type)  │   │
│              │                  │ ├──────────────────────────┤   │
│              │                  │ │ webhook receiver         │   │
│              │                  │ │ (HTTP POST, webhook type)│   │
│              │                  │ └──────────────────────────┘   │
└──────┬───────┴────────┬─────────┴────────────────────────────────┘
       │                │
       ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PrivOS Sandbox│  │   MongoDB    │  │    MinIO      │
│  (Claude Code│  │  (users,     │  │  (context +   │
│   instance)  │  │   rooms,     │  │   skill files)│
│              │  │   triggers)  │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
```

This diagram is the **PrivOS Sandbox runtime** (`customFields.agentRuntime.kind`
absent or `'sandbox'`). A bot with `kind: 'harness'` replaces the "PrivOS
Sandbox HTTP Client" box with a WSS relay to an operator-owned bridge CLI
instead — same dispatch call site, same reply contract; see
[Agent Harness Runtime](./agent-harness-runtime.md).

## Data Flow: User Message → Agent Reply

```
User types message in agent room
        │
        ▼
afterSaveMessage callback
        │
        ├─ room.customFields.isAgentRoom? → No → skip
        │
        ▼ Yes
handleAgentRoomMessage()
        │
        ├─ Skip if message from bot (unless t='agent-trigger')
        ├─ Skip if empty or system message
        ├─ Per-room concurrency guard (1 reply at a time)
        │
        ▼
getContext(roomId)  ← reads IDENTITY.md + CLAUDE.md from MinIO (5min cache)
        │
        ▼
writeClaudeMdToProjectPath()  ← writes to /tmp/privos-sandbox/{roomId}/CLAUDE.md
        │
        ▼
BotMessageService.startStreaming()  ← creates placeholder message
        │
        ▼
streamResponse() → PrivOS Sandbox /api/attempts
        │
        ├─ onTextDelta → BotMessageService.streamChunk() (150ms throttle)
        ├─ onToolActivity → show tool hint
        ├─ onComplete → BotMessageService.endStreaming()
        └─ onError → error message + endStreaming()
```

## Data Flow: Trigger → Agent Reply

```
Trigger source (cron / webhook / event)
        │
        ▼
injectTriggerMessage()
        │
        ├─ Resolve agent room: agent-room-{agentId}
        ├─ Resolve bot user
        ├─ Compose message: prompt + optional context
        │
        ▼
sendMessage(botUser, { rid, msg, t: 'agent-trigger', triggerId })
        │
        ▼
afterSaveMessage callback fires
        │
        ▼
handleAgentRoomMessage()
        │
        ├─ message.u._id === botId BUT t === 'agent-trigger' → ALLOWED
        │
        ▼
(same flow as user message: Sandbox → stream → reply)
```

## Component Map

### Server

| File | Purpose |
|------|---------|
| `server/configuration/bot.ts` | Startup: registers callbacks + heartbeat cron |
| `server/services/privos-sandbox-agent-service.ts` | HTTP client for PrivOS Sandbox `/api/attempts` |
| `server/services/agent-room-reply-handler.ts` | Message → Sandbox → streamed reply |
| `server/services/agent-context-cache.ts` | MinIO file reader with 5-min LRU cache |
| `server/services/agent-context-file-generator.ts` | Generates IDENTITY.md, MEMORY.md, CLAUDE.md |
| `server/services/agent-trigger-injector.ts` | `injectTriggerMessage()` — shared funnel for all trigger types |
| `server/services/agent-trigger-skill-generator.ts` | Generates skill .env, SKILL.md, and 5 JS scripts |
| `server/services/agent-event-trigger-handler.ts` | Dispatches RC events to matching agent triggers |
| `server/cron/agentHeartbeat.ts` | Every-60s cron evaluating cron triggers |
| `app/api/server/v1/agents.ts` | Agent builder chat, agent creation, TS type declarations |
| `app/api/server/v1/agent-trigger-endpoints.ts` | Trigger CRUD endpoints + webhook receiver |

### Client

| File | Purpose |
|------|---------|
| `client/sidebar/header/CreateBotModal.tsx` | Tabbed modal: Bot / Agent tabs |
| `client/sidebar/header/agent-builder/*` | Agent builder chat interface |
| `client/views/room/agent-settings/AgentSettingsTab.tsx` | Main agent settings tab content |
| `client/views/room/agent-settings/AgentTriggerForm.tsx` | Add trigger form (type-conditional fields) |
| `client/views/room/Header/RoomHeader.tsx` | Passes `isAgentRoom` to RoomTabs |
| `client/views/room/Header/RoomTabs.tsx` | Renders Agent Settings tab for agent rooms |
| `client/views/room/providers/RoomToolboxProvider.tsx` | Registers `agent-settings` main content tab |
| `client/views/room/Room.tsx` | Full-layout mode for agent settings |

## MongoDB Schema

### Bot User Document (users collection)

```js
{
  _id: "bot-user-id",
  type: "bot",
  username: "my-agent",
  name: "My Agent",
  active: true,
  roles: ["bot"],
  _createdBy: "owner-user-id",
  customFields: {
    isAgentBot: true,
    agentData: {
      name: "My Agent",
      username: "my-agent",
      purpose: "Help with customer support",
      personality: "Friendly and professional",
      knowledge: ["customer service", "product FAQ"],
      instructions: "Always greet the user first"
    },
    agentTriggers: [
      {
        id: "random-id",
        type: "cron",           // "cron" | "webhook" | "event"
        enabled: true,
        schedule: "every_1h",   // cron only
        prompt: "Scan trending news",
        lastRunAt: ISODate("2026-04-16T10:00:00Z")
      },
      {
        id: "random-id-2",
        type: "webhook",
        enabled: true,
        prompt: "Process incoming data",
        webhookToken: "unique-token",
        webhookSecret: "hmac-secret",
        lastRunAt: null
      },
      {
        id: "random-id-3",
        type: "event",
        enabled: true,
        event: "message.new",
        sourceRoomId: "GENERAL",  // empty = all rooms
        promptTemplate: "Triage this support message",
        lastRunAt: null
      }
    ]
  }
}
```

### Agent Room Document (rooms collection)

```js
{
  _id: "agent-room-{botId}",
  t: "p",                      // private room
  name: "Agent - My Agent",
  customFields: {
    isAgentRoom: true,
    agentBotId: "bot-user-id",
    agentBotUsername: "my-agent"
  }
}
```

## External Dependencies

| Service | Purpose | Config |
|---------|---------|--------|
| **PrivOS Sandbox** | Claude Code AI backend | `Admin > Bots > PrivOS Sandbox` or env `PRIVOS_SANDBOX_URL` + `PRIVOS_SANDBOX_API_KEY` |
| **MinIO** | File storage for context/skill files | Shared MinIO instance (existing) |
| **MongoDB** | User docs (triggers), room docs | Shared MongoDB (existing) |
| **Agenda** (`@rocket.chat/cron`) | Cron job scheduling | MongoDB-backed, handles multi-instance locking |
