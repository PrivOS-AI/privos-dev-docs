# Agent Rooms

Each agent lives in a dedicated private room. Messages in the room are forwarded to PrivOS Sandbox, and responses are streamed back as bot messages.

## Room Structure

| Field | Value |
|-------|-------|
| `_id` | `agent-room-{botId}` |
| `t` | `p` (private) |
| `name` | `Agent - {AgentName}` |
| `customFields.isAgentRoom` | `true` |
| `customFields.agentBotId` | Bot user ID |
| `customFields.agentBotUsername` | Bot username |

**Members:** Room creator (owner) + bot user (owner role).

## Auto-Reply Handler

File: `server/services/agent-room-reply-handler.ts`

Registered as an `afterSaveMessage` callback in `server/configuration/bot.ts`. Fires for every message saved in a room where `customFields.isAgentRoom === true`.

### Guard Conditions

| Condition | Action |
|-----------|--------|
| `!room.customFields.agentBotId` | Skip |
| Message from bot AND `t !== 'agent-trigger'` | Skip (loop prevention) |
| Empty message | Skip |
| System message AND `t !== 'agent-trigger'` | Skip |
| Room already has active reply | Skip (concurrency guard) |

The `t === 'agent-trigger'` exception allows trigger-injected messages (sent by the bot itself) to be processed by the reply handler.

### Concurrency

One reply at a time per room. A `Set<string>` tracks active rooms. If a room is already processing, new messages are silently dropped. The set entry is cleared in a `finally` block.

### Reply Pipeline

1. **Read context** — `getContext(roomId)` reads `IDENTITY.md` and `CLAUDE.md` from MinIO (5-min LRU cache, max 500 entries)
2. **Write CLAUDE.md to disk** — `writeClaudeMdToProjectPath(roomId, claudeMd)` writes to `/tmp/privos-sandbox/{roomId}/CLAUDE.md` so PrivOS Sandbox reads it as project config
3. **Start streaming** — `BotMessageService.startStreaming()` creates a placeholder bot message
4. **Call PrivOS Sandbox** — `streamResponse()` → `syncResponse()` → `POST /api/attempts` with:
   - `prompt`: user message (or trigger prompt + context)
   - `systemContext`: IDENTITY.md content
   - `projectId`: roomId
   - `projectRootPath`: `/tmp/privos-sandbox/{roomId}`
5. **Stream chunks** — `onTextDelta` → `BotMessageService.streamChunk()` (throttled at 150ms intervals)
6. **Finalize** — `onComplete` → `BotMessageService.endStreaming()` with full text

### Error Handling

If PrivOS Sandbox errors, the bot sends: *"Sorry, I encountered an error processing your message. Please try again."*

## Engaging an Agent Outside Its Room (mentions & replies)

Mention and thread replies are room-visible, so they always answer with the bot's own access —
isolated lists are never read as the mentioning user there (see `DELEGATED_READ_AS_ASKER.md`).

In shared rooms where an agent bot is a member (`server/configuration/bot.ts`), the agent
is engaged when a message addresses it. Three address forms are equivalent and pass
through the same permission gates:

1. **@-mention** — the message mentions the agent bot.
2. **Quote reply** — the message quotes one of the agent's messages (the quoted author is
   resolved from the quote attachment's `message_link`).
3. **Thread reply** — the message is posted in a thread whose root message the agent
   authored (covers agent broadcasts that have no ownership record).

A message the agent itself authored never re-engages it.

## Sandbox mode: collocated vs dedicated

A room's agent runs in one of two sandbox modes, chosen per room (proxy-persisted project `mode`):

| Mode | Container | Built-in tools | Use |
|---|---|---|---|
| **collocated** (default on tenants seeded with collocation) | shared VM core for many rooms/bots | `Read`, `Write`, `Edit`, `Glob`, `Grep`, `Skill` only — `Bash`, `Task*` and `mcp__*` are **denied by policy** | cheap chat agents without shell skills |
| **dedicated** | one container per `(room, bot)` | full set incl. `Bash`/`Task*` | agents whose skills run scripts (the hub `privos-list` / `privos-assistant` skills need `Bash`), and delegated reads (`DELEGATED_READ_AS_ASKER.md`) |

Symptoms of a skill room left collocated: the agent reports "no shell / Bash tool available", and
the AI Chat window shows the `dedicated-mode-required` notice. Switching is an operator action on
the tenant sandbox proxy (`POST /api/sandbox/projects/<projectId>/mode {"mode":"dedicated"}` then
`/start`); the hub sends the room's `collocated` directive on every dispatch.

## Context Files

Uploaded to MinIO during agent creation. PrivOS Sandbox auto-syncs files from MinIO using `projectId = roomId`.

### IDENTITY.md

Agent identity document used as `systemContext` in Sandbox requests.

```markdown
# Agent Name

## Purpose
Help with customer support

## Personality
Friendly and professional

## Knowledge Areas
- Product FAQ
- Billing
```

### CLAUDE.md

Project-level config that PrivOS Sandbox reads from CWD.

```markdown
# CLAUDE.md

## Role
You are Support Agent.

## Purpose
Help with customer support

## Instructions
Always greet users by name

## Skills
You have self-management skills in `.claude/skills/privos-agent-management/`.
Read `SKILL.md` in that folder for trigger management capabilities.

## Rules
- Stay in character at all times
- Be helpful, accurate, and concise
- If unsure, say so honestly
- Respect user privacy
```

### MEMORY.md

Empty initially. The agent builds memory over conversations.

```markdown
# MEMORY

(Agent builds memory over conversations)
```

## Context Cache

File: `server/services/agent-context-cache.ts`

In-memory LRU cache for MinIO file reads.

| Setting | Value |
|---------|-------|
| TTL | 5 minutes |
| Max entries | 500 |
| Eviction | Oldest entry when over limit |
| Invalidation | `invalidate(roomId)` clears specific room |

Cache reads `IDENTITY.md` and `CLAUDE.md` in parallel via `Promise.all`.

## PrivOS Sandbox Service

File: `server/services/privos-sandbox-agent-service.ts`

HTTP client for PrivOS Sandbox `/api/attempts` endpoint.

### Configuration

| Source | Setting |
|--------|---------|
| Admin UI | `Admin > Bots > PrivOS Sandbox URL` / `PrivOS Sandbox API Key` |
| Env vars | `PRIVOS_SANDBOX_URL` / `PRIVOS_SANDBOX_API_KEY` |

### Request Payload

```json
{
  "prompt": "user message or trigger prompt",
  "force_create": true,
  "projectId": "agent-room-{botId}",
  "projectName": "Agent - Name",
  "taskId": "agent-room-{botId}",
  "taskTitle": "Agent - Name",
  "projectRootPath": "/tmp/privos-sandbox/{roomId}",
  "request_method": "sync"
}
```

### Response Parsing

The service handles multiple response formats from PrivOS Sandbox:
- Direct `result` or `response` string
- `formatted_data` — JSON string containing array of events
- Anthropic-style `message.content` blocks
- Event arrays with `content_block_delta` and `text_delta`

### Timeouts

Request timeout: 5 minutes (`REQUEST_TIMEOUT_MS`). Uses `AbortController` for cancellation.
