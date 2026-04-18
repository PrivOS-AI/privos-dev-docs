# Agent Settings UI

Dedicated room tab for managing agent triggers. Visible only in agent rooms, pinned alongside Chat and Files tabs.

## Tab Registration

### Room Tabs Bar (for agent rooms)

```
├── Chat (default)
├── Files
└── Agent Settings (pinned, cog icon) ← NEW
```

The tab is:
- **Visible only** when `room.customFields.isAgentRoom === true`
- **Pinned** — always visible, cannot be closed or reordered
- **Full layout** — replaces the message area (same as MCP App tab pattern)

### Component Chain

```
RoomHeader.tsx
  └─ passes isAgentRoom, isAgentSettingsActive, onAgentSettingsClick to:
      RoomTabs.tsx
        └─ renders Agent Settings tab (cog icon) when isAgentRoom
            └─ onClick → openMainContentTab('agent-settings')

RoomToolboxProvider.tsx
  ��─ resolves 'agent-settings' → { tabComponent: AgentSettingsTab }

Room.tsx
  └��� when mainContentTab.id === 'agent-settings' → full layout mode
      └─ createElement(AgentSettingsTab)
```

## AgentSettingsTab

File: `client/views/room/agent-settings/AgentSettingsTab.tsx`

Main container component. Reads `botId` from `room.customFields.agentBotId`.

### Features

- **Trigger list** — fetches from `GET /v1/agents.triggers.list`
- **Type icons** — clock (cron), permalink (webhook), lightning (event)
- **Schedule labels** — human-readable (e.g., "Every hour" instead of "every_1h")
- **Prompt preview** — truncated with ellipsis
- **Last run timestamp** — `toLocaleString()` if available
- **Toggle switch** — enable/disable via `POST /v1/agents.triggers.update`
- **Copy webhook URL** — clipboard copy for webhook triggers
- **Delete button** — remove via `POST /v1/agents.triggers.remove`
- **Add Trigger button** — opens inline form (disabled when 5 triggers reached)
- **Max triggers callout** — warning when limit reached

### Data Fetching

Uses React Query (`@tanstack/react-query`):

```ts
useQuery({
  queryKey: ['agent-triggers', botId],
  queryFn: () => listTriggers({ botId }),
  enabled: !!botId,
})
```

Mutations invalidate the query to refetch after toggle/delete/add.

## AgentTriggerForm

File: `client/views/room/agent-settings/AgentTriggerForm.tsx`

Inline form for adding new triggers. Shows conditional fields based on selected type.

### Type Selector

| Type | Label | Fields shown |
|------|-------|-------------|
| `cron` | Cron (scheduled) | Schedule dropdown + Prompt |
| `webhook` | Webhook (external) | Prompt only (token auto-generated) |
| `event` | Event (internal) | Event dropdown + Source Room ID + Prompt |

### Schedule Options

| Value | Label |
|-------|-------|
| `every_5m` | Every 5 minutes |
| `every_15m` | Every 15 minutes |
| `every_30m` | Every 30 minutes |
| `every_1h` | Every hour |
| `every_6h` | Every 6 hours |
| `every_12h` | Every 12 hours |
| `every_24h` | Every 24 hours |

### Event Options

| Value | Label |
|-------|-------|
| `message.new` | New message |
| `message.mention` | Mention |
| `user.joined` | User joined |
| `file.created` | File created |
| `list.item.created` | List item created |
| `list.item.stage_changed` | List item stage changed |

### Webhook Auto-Copy

On successful webhook trigger creation, the returned `webhookUrl` is automatically copied to clipboard.

### Validation

- Prompt field is required (submit disabled when empty)
- Server-side validation returns error messages displayed inline

## i18n Keys

| Key | Value |
|-----|-------|
| `Agent_Settings` | Agent Settings |
| `Agent_Triggers` | Agent Triggers |
| `Add_Trigger` | Add Trigger |
| `Trigger_Type` | Trigger Type |
| `Trigger_Prompt` | What should the agent do? |
| `Trigger_Schedule` | Schedule |
| `Trigger_Event` | Event |
| `Trigger_Source_Room` | Source Room (optional — all rooms if empty) |
| `Copy_Webhook_URL` | Copy Webhook URL |
| `No_triggers_configured` | No triggers configured yet |
| `Max_triggers_reached` | Maximum 5 triggers per agent |
