# Bot Key Push & Agent Switching

How a chat room hands its bot's API token to PrivOS Sandbox, how the AI chat
re-validates the push when the user switches agents, and how the brain task is
kept in sync across agent changes.

## Why This Exists

PrivOS Sandbox runs as a separate process (claude-ws). Skills and triggers
running inside a brain project authenticate back to chat as the room bot via
`Authorization: Bearer privos_<userId>_<secret>`. To do that, the bot's token
must be stored in the project's `.env` on the brain side. The "Push bot key to
PrivOS Sandbox" CTA in the AI chat input writes that file.

Each `(room, bot, brain)` triple has its own push record. Switching agents in
the AI chat selector targets a different bot, so the overlay must
re-evaluate per-bot — not per-room.

## Components

| File | Role |
|------|------|
| `apps/meteor/server/services/privos-brain-bot-key-service.ts` | `getBotKeyStatus`, `pushBotKeyToBrain` — DB read/write + outbound POST to brain |
| `apps/meteor/app/api/server/v1/agent-privos-brain-bot-key.ts` | REST routes `agents.brain.botKeyStatus` (GET) and `agents.brain.pushBotKey` (POST) |
| `apps/meteor/server/models/BotPrivOSBrainKey.ts` | Mongo collection — stores `sha256(token)` per `(botId, roomId, privosBrainId)` |
| `apps/meteor/client/hooks/aiChat/useBotPrivOSBrainKeyStatus.ts` | React Query hook — status polling + push mutation |
| `apps/meteor/client/components/AIChatBox/ChatBoxInput.tsx` | Renders the overlay CTA, dismissal state, agent selector wiring |
| `claude-ws/src/app/api/bot-key/route.ts` | Brain receiver — writes `.env` and triggers MinIO sync |

## Push Flow

1. Client `POST /v1/agents.brain.pushBotKey` with `{ roomId, botId? }`.
2. Route validates: caller has room access + `edit-room` + (bot owner or `edit-bot` permission).
3. `resolveTargetBotId(roomId, botId)` — if `botId` provided, validates the bot exists, is `type: 'bot'`, and has a subscription to the room. Otherwise falls back to the room's default bot.
4. `pushBotKeyToBrain` reads the bot's active token (no minting — uses the existing one from `BotTokens`), computes `projectId` (agent-room bots use `roomId`; generic rooms use legacy `Agent-{roomId}-{botId}`), and POSTs to `${brainUrl}/api/bot-key` with `x-api-key` header.
5. Brain writes `.env` to `data/projects/{projectId}/` with `PRIVOS_URL`, `PRIVOS_BOT_KEY`, `PRIVOS_BOT_ID`, `PRIVOS_ROOM_ID`, `PRIVOS_PROJECT_ID`.
6. Chat stores `sha256(botToken)` in `BotPrivOSBrainKeys` so the next status check can detect token rotation without leaking the token.

## Status Endpoint

`GET /v1/agents.brain.botKeyStatus?roomId=…&botId=…` returns:

```jsonc
{
  "pushed": false,           // sha256(currentToken) === record.hash AND status === 'success'
  "hasBot": true,            // a bot was resolvable for the room+botId pair
  "hasBrain": true,          // the room has a PrivOS Sandbox configured
  "canPush": true,           // caller has the right permissions
  "status": "success",       // last push outcome
  "pushedAt": "2026-04-29T…",
  "privosBrainId": "thanh-3000.roxane.one",
  "errorMessage": "…"        // on failed pushes
}
```

`pushed` re-becomes `false` when the bot's token rotates (hash mismatch),
when the brain config changes (different `privosBrainId`), or when the user
switches to a bot that hasn't pushed to that brain yet.

## Agent Selector Re-validation

The hook keys its query on `(roomId, botId)`:

```ts
useBotPrivOSBrainKeyStatus({ roomId, botId: selectedAgent?.botUserId });
```

Switching agents triggers a fresh fetch instead of returning the cached default-bot status. Dismissal state is keyed by `${roomId}:${botId}` so dismissing the CTA for one agent doesn't hide it after switching to another. Push polling continues every 15s, on mount, and on window focus while the chat is open.

## Mid-Session Context Handover

When the user switches agents *inside* an existing session, the new bot's
brain task has zero conversation history. `buildContextForStreaming`
(`apps/meteor/app/agent-chat/server/contextCompaction.ts`) detects this:

```ts
const agentChanged = !!lastBotId && lastBotId !== botId;
```

When `agentChanged` is true, it bypasses resume and delta modes and pushes
`summary + activeMessages` to the new agent on its first turn. After the
stream completes, the new bot's per-scope track is written
(`scope = ${roomId}:${botId}`) and `lastBotId` is updated. Subsequent turns
with the same agent fall back to resume mode normally.

If the user later switches **back** to a previous agent, `agentChanged` fires
again — that brain task missed the messages exchanged with the other agent
in the meantime, so a full push catches it up.

## Last-Agent Restoration on Re-open

`AIChatBox.tsx` resolves the agent on mount with this priority:

1. `session.lastBotId` — set server-side after each successful round-trip
   (`AIChatSessions.updateLastBotId`). Cross-device, persists in DB,
   pinned per session.
2. `aiChatBotStorage.getLastBot(userId, roomId)` — localStorage fallback,
   updated on every selector change. Single browser, room-scoped. Used when
   the session has no `lastBotId` yet (brand-new conversation).
3. The room's `isDefault` bot.

The effect runs whenever `session?.lastBotId` changes, so opening a different
session re-resolves the agent.

## Authorization for Bot Self-Management

Brain skills call back as the bot itself. For trigger management
(`agents.triggers.*` only), `verifyBotOwnership` accepts:

1. The bot's `_createdBy` user.
2. Anyone with `owner` or `leader` role on the bot's `customFields.agentRoomId`. The bot is provisioned with `owner` on its agent room (`agents.ts:462`), so a bot using its own bearer token passes this check.
3. Server admins (`view-user-administration`).

Other endpoints that consume the same bearer token (`agents.brain.*`,
`agents.reply`, etc.) keep their original auth checks unchanged.

## Failure Modes

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| 401 on bot-bearer call to chat | Token rotated since last push | Press "Push bot key to PrivOS Sandbox" again |
| 401 on bot-bearer call to chat | Bot user disabled / `active: false` token | Re-provision via Agent Builder |
| 400 `bot-not-room-default` | (legacy) Bot lacked `customFields.botDefaultOfRoomId === roomId` | Now relaxed: any bot subscribed to the room is valid |
| 400 `Bot not found or not authorized` from `agents.triggers.*` | Caller is the bot itself but isn't owner/leader of the agent room | Verify the bot's subscription has `owner` role |
| `hasBrain: false` despite room config | Room's `privosBrain.url` not set / not enabled | Configure via room settings |
| Mid-session agent switch produces blank-context replies | Brain task created from scratch but only got the latest turn | Fixed — `buildContextForStreaming` now detects `agentChanged` and force-pushes full history |

## Related Docs

- [Architecture](./architecture.md) — high-level system map
- [Agent Rooms](./agent-rooms.md) — room provisioning, default bot custom fields
- [Trigger API Reference](./trigger-api-reference.md) — `agents.triggers.*` endpoints
- [Self-Management Skills](./self-management-skills.md) — how skills authenticate back to chat
