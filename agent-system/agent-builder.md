# Agent Builder

Conversational flow for creating AI agents. Users chat with a builder assistant that gathers agent details, then the system provisions the full agent infrastructure.

## Entry Point

**Sidebar → Create Bot Modal → "Agent" tab**

The `CreateBotModal` has two tabs:
- **Bot** — traditional bot creation (username, token, webhook)
- **Agent** — conversational builder powered by PrivOS Sandbox

## Builder Chat Flow

```
User opens Agent tab
        │
        ▼
AgentBuilderChat component
        │  (manages sessionId, history[], agentData state)
        │
        ▼
User types message → POST /v1/agents.builderChat
        │  { message, sessionId, history, timezone }
        │
        ▼
Server builds full prompt (history + current message)
        │
        ▼
syncResponse() → PrivOS Sandbox /api/attempts
        │  systemContext = AGENT_BUILDER_SYSTEM_PROMPT + current-time line
        │  projectId = sessionId (UUID)
        │
        ▼
Response returned to client
        │
        ├─ Contains ```json { "agentReady": true, "agentData": {...} }``` ?
        │  → Yes: extract agentData, show Create Agent button
        │  → No: display as chat message, continue conversation
```

### Builder System Prompt

The builder assistant asks about (one at a time):
1. Agent **name** (required)
2. Agent **purpose/role**
3. **Personality** and tone
4. **Knowledge areas** / expertise
5. Specific **instructions** or rules
6. Whether the agent should run on a **schedule** (reports, periodic checks, reminders)

When enough info is gathered, it outputs a JSON block (`triggers` is optional, see [Schedules](#schedules)):

```json
{
  "agentReady": true,
  "agentData": {
    "name": "Support Agent",
    "username": "support-agent",
    "purpose": "Handle customer support inquiries",
    "personality": "Friendly and professional",
    "knowledge": ["product FAQ", "billing", "troubleshooting"],
    "instructions": "Always greet users by name",
    "triggers": [
      { "type": "cron", "schedule": "0 9 * * *", "timezone": "Asia/Bangkok", "prompt": "Post the morning digest" }
    ]
  }
}
```

### Schedules

The schedule rules are one constant, `AGENT_BUILDER_SCHEDULE_RULES` in `apps/meteor/app/api/server/v1/agents.ts`, used by
both builder prompts. A trigger's `schedule` is exactly one of:

- a preset key: `every_1m`, `every_2m`, `every_3m`, `every_4m`, `every_5m`, `every_15m`, `every_30m`, `every_1h`,
  `every_6h`, `every_12h`, `every_24h`;
- a 5-field cron expression, for example `0 */3 * * *` (every 3 hours), `*/10 * * * *` or `30 9 * * 1-5` (weekdays at
  09:30). Whenever the schedule names a wall-clock time the trigger also carries `"timezone"` with the user's IANA name;
  a plain interval never does;
- a one-time run `at:<ISO-8601 with offset or Z>`, for "once", "tomorrow at 6pm" or "in 30 minutes", written in the user's
  offset (for example `at:2026-10-11T18:00:00+07:00`). A one-time run has no `timezone`.

The model is told what time it is for the user: `agents.builderChat` appends one line to the system context, built by
`builderTimeContext` (`server/lib/agent-builder-time-context.ts`):

```
Current time: 2026-10-11T11:30:00+07:00 (Asia/Bangkok). Write one-time instants in this offset.
```

The zone is the `timezone` the client sent (the browser's IANA name, from `use-agent-builder-chat.ts`) when it is valid;
otherwise the user's stored `utcOffset` rendered as `UTC±HH:MM`; otherwise UTC. The model copies the offset instead of
converting. The hub never parses natural language, so the review step shows the resolved label ("Once at <local date
time>", "Daily at 09:00 (Asia/Bangkok)") before the user confirms.

On `agents.create`, builder triggers pass through `checkSchedule` (`cron-schedule-utils.ts`): only cron triggers with a
prompt are kept (prompt cut to 500 characters, at most 5 triggers), a schedule the hub would refuse or a one-time instant
already past is dropped silently, and `timezone` is kept only on a recurring cron when it is a valid IANA name. Grammar and
messages: [Trigger API Reference](./trigger-api-reference.md#schedules). A zip import restores triggers through the
dedicated restorer instead, which keeps `nextAction`, `handler` and `timezone` and warns about what it skips.

## Agent Creation

When user clicks "Create Agent", the client calls `POST /v1/agents.create`.

### What the server provisions

1. **Bot user** — `type: 'bot'`, `roles: ['bot']`, `customFields.isAgentBot: true`
2. **Bot API token** — auto-generated via `BotTokenService.createToken()`
3. **Agent room** — private room `agent-room-{botId}` with `customFields.isAgentRoom: true`
4. **Bot as room owner** — full permissions in agent room
5. **Context files** (uploaded to MinIO → synced to PrivOS Sandbox CWD):
   - `IDENTITY.md` — name, purpose, personality, knowledge areas
   - `MEMORY.md` — empty memory index (builds over time)
   - `CLAUDE.md` — behavior rules, instructions, skills reference
6. **Skill files** (uploaded to MinIO → synced to PrivOS Sandbox CWD):
   - `.claude/skills/privos-agent-management/.env` — bot ID, token, chat URL
   - `.claude/skills/privos-agent-management/SKILL.md` — usage docs
   - 5 JS scripts: `trigger-list.js`, `trigger-add.js`, `trigger-update.js`, `trigger-remove.js`, `trigger-run.js`
7. **Welcome message** — bot sends intro message in agent room
8. **Default avatar** — `avatars/default-bot-avatar.png`

### Rollback

If room creation fails after bot user is created, the bot user is deleted (cleanup).

## API Endpoints

### POST /v1/agents.builderChat

Conversational builder — send message, get AI response.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `message` | string | Yes | User message (max 10,000 chars) |
| `sessionId` | string | Yes | UUID v4 session identifier |
| `history` | array | No | Previous messages `[{role, content}]` (max 100) |
| `timezone` | string | No | The user's IANA timezone (the browser's); an invalid value falls back to the account's UTC offset |

**Response:** `{ response: string }`

### POST /v1/agents.create

Provision the full agent.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Display name (2-200 chars) |
| `username` | string | Yes | Username (`[a-zA-Z0-9._-]+`) |
| `agentData` | object | Yes | `{ name, username, purpose, personality?, knowledge?, instructions? }` |

**Response:**
```json
{
  "bot": { "_id": "...", "username": "...", "name": "..." },
  "room": { "_id": "agent-room-xxx", "name": "...", "fname": "Agent - Name" },
  "token": "privos_xxx_...",
  "message": "Agent created successfully"
}
```

**Requires permission:** `create-bot`

### Field Size Limits

| Field | Max Length |
|-------|-----------|
| `name` | 200 chars |
| `username` | 100 chars |
| `purpose` | 2,000 chars |
| `personality` | 2,000 chars |
| `knowledge` | 20 items, 500 chars each |
| `instructions` | 5,000 chars |
