# Agent Builder

Conversational flow for creating AI agents. Users chat with a builder assistant that gathers agent details, then the system provisions the full agent infrastructure.

## Entry Point

**Sidebar → Create Bot Modal → "Agent" tab**

The `CreateBotModal` has two tabs:
- **Bot** — traditional bot creation (username, token, webhook)
- **Agent** — conversational builder powered by Privos Brain

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
        │  { message, sessionId, history }
        │
        ▼
Server builds full prompt (history + current message)
        │
        ▼
syncResponse() → Privos Brain /api/attempts
        │  systemContext = AGENT_BUILDER_SYSTEM_PROMPT
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

When enough info is gathered, it outputs a JSON block:

```json
{
  "agentReady": true,
  "agentData": {
    "name": "Support Agent",
    "username": "support-agent",
    "purpose": "Handle customer support inquiries",
    "personality": "Friendly and professional",
    "knowledge": ["product FAQ", "billing", "troubleshooting"],
    "instructions": "Always greet users by name"
  }
}
```

## Agent Creation

When user clicks "Create Agent", the client calls `POST /v1/agents.create`.

### What the server provisions

1. **Bot user** — `type: 'bot'`, `roles: ['bot']`, `customFields.isAgentBot: true`
2. **Bot API token** — auto-generated via `BotTokenService.createToken()`
3. **Agent room** — private room `agent-room-{botId}` with `customFields.isAgentRoom: true`
4. **Bot as room owner** — full permissions in agent room
5. **Context files** (uploaded to MinIO → synced to Privos Brain CWD):
   - `IDENTITY.md` — name, purpose, personality, knowledge areas
   - `MEMORY.md` — empty memory index (builds over time)
   - `CLAUDE.md` — behavior rules, instructions, skills reference
6. **Skill files** (uploaded to MinIO → synced to Privos Brain CWD):
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
