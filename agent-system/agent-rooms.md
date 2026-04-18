# Agent Rooms

Each agent lives in a dedicated private room. Messages in the room are forwarded to Privos Brain, and responses are streamed back as bot messages.

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
2. **Write CLAUDE.md to disk** — `writeClaudeMdToProjectPath(roomId, claudeMd)` writes to `/tmp/privos-brain/{roomId}/CLAUDE.md` so Privos Brain reads it as project config
3. **Start streaming** — `BotMessageService.startStreaming()` creates a placeholder bot message
4. **Call Privos Brain** — `streamResponse()` → `syncResponse()` → `POST /api/attempts` with:
   - `prompt`: user message (or trigger prompt + context)
   - `systemContext`: IDENTITY.md content
   - `projectId`: roomId
   - `projectRootPath`: `/tmp/privos-brain/{roomId}`
5. **Stream chunks** — `onTextDelta` → `BotMessageService.streamChunk()` (throttled at 150ms intervals)
6. **Finalize** — `onComplete` → `BotMessageService.endStreaming()` with full text

### Error Handling

If Privos Brain errors, the bot sends: *"Sorry, I encountered an error processing your message. Please try again."*

## Context Files

Uploaded to MinIO during agent creation. Privos Brain auto-syncs files from MinIO using `projectId = roomId`.

### IDENTITY.md

Agent identity document used as `systemContext` in Brain requests.

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

Project-level config that Privos Brain reads from CWD.

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

## Privos Brain Service

File: `server/services/privos-brain-agent-service.ts`

HTTP client for Privos Brain `/api/attempts` endpoint.

### Configuration

| Source | Setting |
|--------|---------|
| Admin UI | `Admin > Bots > Privos Brain URL` / `Privos Brain API Key` |
| Env vars | `PRIVOS_BRAIN_URL` / `PRIVOS_BRAIN_API_KEY` |

### Request Payload

```json
{
  "prompt": "user message or trigger prompt",
  "force_create": true,
  "projectId": "agent-room-{botId}",
  "projectName": "Agent - Name",
  "taskId": "agent-room-{botId}",
  "taskTitle": "Agent - Name",
  "projectRootPath": "/tmp/privos-brain/{roomId}",
  "request_method": "sync"
}
```

### Response Parsing

The service handles multiple response formats from Privos Brain:
- Direct `result` or `response` string
- `formatted_data` — JSON string containing array of events
- Anthropic-style `message.content` blocks
- Event arrays with `content_block_delta` and `text_delta`

### Timeouts

Request timeout: 5 minutes (`REQUEST_TIMEOUT_MS`). Uses `AbortController` for cancellation.
