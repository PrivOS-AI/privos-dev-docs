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

## Messages during a reply (mid-turn steering policy)

When a new message reaches an agent bot while it is still generating a reply in
the **same session** (same thread / AI-chat window), its behavior is governed by
a per-agent policy, editable in Agent Settings via the "When a new message
arrives while replying" select and stored at
`customFields.agentTurnPolicy.onNewMessage` (default `steer`). Read/write through
`GET`/`POST /v1/agents.turnPolicy` (owner or `view-user-administration` only —
never bare `edit-bot`, no bot-self path). The policy is runtime-agnostic (sandbox
and harness agents alike).

| Policy | What the user sees |
|---|---|
| `steer` (default) | No parallel reply. The new message is delivered into the running turn; the agent keeps working and takes it into account. The new message keeps its own bubble but gets no separate reply — it shows "Added to the reply above." |
| `queue` | The new message waits until the current reply finishes, then runs with its own reply. |
| `interrupt` | The current reply is stopped and the new message replaces it; the stopped reply keeps its partial text with "— interrupted, continued below". |
| `owner-interrupt` | Only the bot owner can interrupt; everyone else's message waits. |

Only the **same sender** may steer or interrupt their own running turn; another
member's message always waits (except the owner under `owner-interrupt`).

**Native vs fallback per runtime.** "Steer" and "interrupt" try a native
mid-turn injection first, falling back to cancel-and-remerge when the runtime
can't inject:

| Runtime | Native steer | Fallback |
|---|---|---|
| Sandbox, `privos-agent-sdk` provider | `POST /api/attempts/:id/steer` → `Session.steer()`, drained at the next turn boundary | cancel + merged re-prompt |
| Sandbox, Claude CLI / Codex / Antigravity | none | cancel + merged re-prompt |
| Harness, adapter advertising `_meta.steering.supported` (claude-agent-acp, codex-acp) | `turn.steer` → ACP `_session/steering` → `injected` | cancel + merged re-prompt |
| Harness, other adapters (Cursor, Goose, custom) | none | cancel + merged re-prompt |

The fallback and native paths share the same framing wording, so the agent is
oriented identically either way.

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
