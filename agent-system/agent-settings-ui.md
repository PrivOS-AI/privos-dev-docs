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

### Trigger card

Every trigger renders as one generic card; nothing in it is specific to a routine. Pure helpers in
`client/views/room/agent-settings/trigger-summary.ts` (`describeTriggerWhen`, `playbookPathOf`, `promptFieldOf`,
`handlerNameOf`, `describeLastOutcome`) decide what is shown.

| Line | Content |
|---|---|
| Title | `name`, else the **When** summary, else "Webhook", else the type. A tag reads "Ran at {{time}}" on a one-time trigger that has fired (disabled, with `lastRunAt`); the plain "Last run" line is hidden for it |
| Subtitle | `description` |
| **When** | cron: the schedule label (`cronDisplayLabel`), e.g. "Every 3 hours", "Daily at 09:00 (Asia/Bangkok)" or "Once at <local date time>"; plain event: the event; filtered event: one line `events · subject · rooms · predicates · cooldown · batch`. A wall-clock pattern names its timezone only when one is stored and it differs from the browser's. Empty for a webhook |
| **Then** | `Agent`, or `handler <name> → agent on exit 10` for a `run_handler` trigger. Hidden for an `emit_event` webhook |
| **Playbook** | the first `AgentFiles/routines/<name>.md` path found in the trigger's prompt field, as text |
| Prompt | the full prompt of the trigger's type (`promptTemplate` for event, `prompt` otherwise) |
| Next run | a pure interval (from the last run) or the instant of a pending one-time trigger |
| Last run | `lastRunAt`. For a trigger **without** a handler, ` · ran, nothing to say` (`quiet`) or ` · ran, posted a reply` (`posted`) |
| Last outcome | handler triggers only: `Last outcome: <lastOutcome> (<lastError>)` |
| Hints (red) | `handler stopped by the owner: the agent runs instead` when `lastError` is `owner-disabled`; `paused after 3 failed handler runs: fix the handler or edit the trigger` when `lastOutcome` is `paused` |

The scoped-rooms line is not shown for a filtered trigger (its rooms are in the filter), and the Rooms picker is hidden for
a filtered trigger and for a cron whose Then is a handler.

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
| `cron` | Cron (scheduled) | Schedule dropdown + Then + Prompt |
| `webhook` | Webhook (external) | Then + Prompt (token auto-generated) |
| `event` | Event (internal) | Event dropdown + Source Room ID + Then + Prompt |

### Then (what runs)

A **Then** select appears for all three types: `Agent turn` or `Run handler` (a webhook also keeps `Emit Event`). Choosing
`Run handler` shows a **Handler name** input (placeholder `e.g. screen-rooms`) and the hint "A service of this agent's room
that is declared as a handler. It runs first: exit 0 ends the run, exit 10 wakes the agent with its output, anything else
runs the agent." For a cron it also shows "A handler runs once per fire, so this cron runs globally and not in rooms." and
hides the Rooms picker. `Run handler` is not offered for a harness agent. The form sends `nextAction: "run_handler"` and
`handler: { name }`.

### Schedule Options

The dropdown of a cron trigger (value, label):

| Value | Label |
|-------|-------|
| `every_1m` … `every_30m` (1, 2, 3, 4, 5, 15, 30) | Every minute, Every 2 minutes, … Every 30 minutes |
| `every_1h` | Every hour |
| `0 */2 * * *`, `0 */3 * * *`, `0 */4 * * *`, `0 */8 * * *` | Every 2 hours, Every 3 hours, Every 4 hours, Every 8 hours |
| `every_6h`, `every_12h`, `every_24h` | Every 6 hours, Every 12 hours, Every 24 hours |
| `once` | Run once at… |
| `custom` | Custom (cron) |

The new hour entries emit a raw cron expression, not a new preset key. Source: `AgentTriggerForm.tsx`.

**Custom (cron)** has two modes. *Schedule builder* offers the patterns "Every X minutes" (number input 1–59), "Every X
hours" (number input 1–23, emits `0 */N * * *`), "Daily at", "Weekly on" and "Monthly on". *Manual cron* is a raw cron text
input (placeholder `*/10 * * * *`). A preview line shows the label of what will be saved.

**Run once at…** shows a native `datetime-local` input whose minimum is now. The value is read in the browser's local time
and saved as `at:<UTC ISO>`; submit stays disabled until the instant is in the future, and a past time shows "Pick a date
and time in the future". A valid value shows the timezone hint and the "Once at …" preview.

**Timezone.** Daily, weekly and monthly patterns send `timezone` (the browser's IANA name, from
`Intl.DateTimeFormat().resolvedOptions().timeZone`) and show the hint "Times are in {{timezone}}". Pure intervals, presets,
manual cron and one-time runs send none, so they follow the server's UTC.

### Inline editor (pencil)

The pencil on a card opens an editor. It sends a field only when it changed, so saving a description never rewrites the
schedule of a wall-clock cron.

| Field | Shown when | Behaviour |
|---|---|---|
| Schedule | cron, a pure interval | "Every N minutes" number input with the hint "1–59 minutes, a whole number of hours (60, 120, …), or a whole number of days (1440, 2880, …)." |
| Schedule | cron, one-time | `datetime-local` input prefilled from the instant, hint "Times are in <browser timezone>", saved as `at:<UTC ISO>` with the same future check |
| Schedule | cron, anything else (wall-clock, steps) | **Cron expression**: a raw text input prefilled with the stored expression, hint "Times are in <trigger timezone or UTC>"; the hub validates it |
| Filter (JSON) | filtered event trigger | a textarea with the normalised filter; sent only when edited; invalid JSON shows "The filter is not valid JSON" and a hub refusal is shown verbatim |
| Then | not an `emit_event` webhook | `Agent turn` or `Run handler` plus the handler name; sent only when changed. Choosing a handler for a cron sends `sourceRoomIds: []` in the same save |
| Rooms | cron without a handler, plain event trigger | the rooms picker |
| Description, Prompt | always | the prompt field is the one of the trigger's type |

The editor does not edit `timezone` of a stored trigger.

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
| `Agent_Trigger_When` | When |
| `Agent_Trigger_Then` | Then |
| `Agent_Trigger_Then_Agent` | Agent |
| `Agent_Trigger_Then_Handler` | handler {{name}} → agent on exit 10 |
| `Agent_Trigger_Playbook` | Playbook |
| `Agent_Trigger_Last_Outcome` | Last outcome |
| `Agent_Trigger_Schedule_Once` | Run once at… |
| `Agent_Trigger_Ran_At` | Ran at {{time}} |
| `Agent_Trigger_Timezone_Hint` | Times are in {{timezone}} |
| `Agent_Trigger_Cron_Raw` | Cron expression |

The other trigger keys (`Agent_Trigger_*`) live in `packages/i18n/src/locales/en.i18n.json`, the only locale that carries them.
