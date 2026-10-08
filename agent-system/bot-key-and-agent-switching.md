# Bot Key Push & Agent Switching

How a chat room hands its bot's API token to PrivOS Sandbox, how the AI chat
re-validates the push when the user switches agents, and how the sandbox task is
kept in sync across agent changes.

## Why This Exists

PrivOS Sandbox runs as a separate process (privos-sandbox). Skills and triggers
running inside a sandbox project authenticate back to chat as the room bot via
`Authorization: Bearer <bot key>`. In sandbox mode (`PRIVOS_SANDBOX_MODE=true`)
the key never enters the VM or the project's `.env`: the sandbox stores it in the
proxy's egress catalog, and the proxy attaches it to the hub routes the catalog
allows. Without sandbox mode (a plain privos-sandbox install with no proxy) the
key stays in the project's `.env`. The "Push bot key to PrivOS Sandbox" CTA in
the AI chat input performs that handover.

Each `(room, bot, sandbox)` triple has its own push record. Switching agents in
the AI chat selector targets a different bot, so the overlay must
re-evaluate per-bot — not per-room.

## Components

| File | Role |
|------|------|
| `apps/meteor/server/services/privos-sandbox-bot-key-service.ts` | `getBotKeyStatus`, `pushBotKeyToSandbox` — DB read/write + outbound POST to sandbox + DDP progress/status events |
| `apps/meteor/server/services/privos-sandbox-bot-key-push-retry.ts` | Classifies failed pushes (transient sandbox failure vs real refusal) and bounds server-side automatic repair |
| `apps/meteor/app/api/server/v1/agent-privos-sandbox-bot-key.ts` | REST routes `agents.sandbox.botKeyStatus` (GET) and `agents.sandbox.pushBotKey` (POST) |
| `apps/meteor/server/models/BotPrivOSSandboxKey.ts` | Mongo collection — stores `sha256(token)` per `(botId, roomId, privosSandboxId)` |
| `apps/meteor/client/hooks/aiChat/useBotPrivOSSandboxKeyStatus.ts` | React Query hook — status polling + push mutation |
| `apps/meteor/client/components/AIChatBox/ChatBoxInput.tsx` | Renders the overlay CTA, dismissal state, agent selector wiring |
| `privos-sandbox/src/app/api/bot-key/route.ts` | Sandbox receiver — writes the project `.env` (ids only in sandbox mode), mirrors it to the proxy, writes the egress catalog rows and triggers MinIO sync |

## Push Flow

1. Client `POST /v1/agents.sandbox.pushBotKey` with `{ roomId, botId? }`.
2. Route validates: caller has room access + `edit-room` + (bot owner or `edit-bot` permission).
3. `resolveTargetBotId(roomId, botId)` — if `botId` provided, validates the bot exists, is `type: 'bot'`, and has a subscription to the room. Otherwise falls back to the room's default bot.
4. `pushBotKeyToSandbox` reads the bot's active token (no minting — uses the existing one from `BotTokens`), computes `projectId` (agent-room bots use `roomId`; generic rooms use legacy `Agent-{roomId}-{botId}`), and POSTs to `${sandboxUrl}/api/bot-key` with `x-api-key` header.
5. Sandbox writes `.env` to `data/projects/{projectId}/` with `PRIVOS_HUB_HOST`, `PRIVOS_BOT_ID`, `PRIVOS_ROOM_ID` and `PRIVOS_PROJECT_ID` (public ids, no secret), mirrors it to the proxy's per-project env, and writes the bot key into the proxy's egress catalog as `platform` rows: hub-host route patterns, project scope, an allowlist (`src/lib/bot-key-egress-catalog.ts` in the sandbox). Only outside sandbox mode does the `.env` also carry `PRIVOS_URL` and `PRIVOS_BOT_KEY`. Every push reconciles the project's platform rows, deleting ones the current build no longer produces; credential-vault rows are never touched.
5a. For a [super agent](./super-agent.md) pushing its own `agent-room-<botId>` project, the payload carries `superAgent: true` while the flag is set and the kill switch is released, and the catalog gains exact routes for room reads and room management. A flag or kill-switch change makes the hub push that project again, so the extra rows follow the state.
6. Chat stores `sha256(botToken)` in `BotPrivOSSandboxKeys` so the next status check can detect token rotation without leaking the token.

## Automatic Repair — Server Is the Sole Initiator

When a room's key is stale (rotation, config change, sandbox state lost), the **server**
starts the repair push itself; the client never initiates an automatic push. Callers that
do trigger an unattended push (e.g. an app via `sandbox:botkey:push`) mark it with
`auto: true` in the `pushBotKey` body. Two rules bound the repair:

- **Only a real refusal bounds it.** A push failure is classified by *what the sandbox
  said*, not its HTTP status (`privos-sandbox-bot-key-push-retry.ts`):
  - **Transient — does not count against the bound:** the sandbox was unreachable, replied
    5xx (the sandbox failing, not refusing), or returned a transient code
    (`project-operation-in-progress`, `envelope-rate-limited`,
    `project-operation-preflight-failed`).
  - **Refusal — counts and bounds the repair:** the sandbox explicitly rejected the key.
- **Progress is visible even for a repair the client did not start** — see the DDP streams
  below.

The manual "Push bot key" CTA and `/push-bot-key` slash command remain for explicit
user-driven pushes.

## Status Endpoint

`GET /v1/agents.sandbox.botKeyStatus?roomId=…&botId=…` returns:

```jsonc
{
  "pushed": false,           // sha256(currentToken) === record.hash AND status === 'success'
  "reason": "hash-mismatch", // why pushed is false: no-record | last-push-failed |
                             // hash-mismatch | config-missing | sandbox-state-lost | sandbox-key-stale
  "hasBot": true,            // a bot was resolvable for the room+botId pair
  "hasSandbox": true,        // the room has a PrivOS Sandbox configured
  "canPush": true,           // caller has the right permissions
  "canAutoPush": true,       // sandbox is configured, so automatic repair can run for any
                             // member — independent of the caller's own push permissions
  "status": "success",       // last push outcome
  "pushedAt": "2026-04-29T…",
  "needsForceOverwrite": true, // present only when the sandbox holds a diverged persona;
                               // pass forceOverwritePersona on pushBotKey to replace it
  "errorMessage": "…"        // on failed pushes
}
```

`privosSandboxId` (the internal board host URL) is intentionally **not** exposed to
clients/apps — `hasSandbox` conveys configured-state without leaking it.

`pushed` re-becomes `false` when the bot's token rotates (hash mismatch),
when the sandbox config changes (different `privosSandboxId`), or when the user
switches to a bot that hasn't pushed to that sandbox yet.

## DDP Streams

Both are room streams (`packages/ddp-client/src/types/streams.ts`):

| Stream | Payload |
|--------|---------|
| `bot-privos-sandbox-key-status-changed` | `{ botId?, reason: 'push' \| 'config-change' \| 'drift', status?: 'success' \| 'failed' \| 'drift', changedAt }` |
| `bot-privos-sandbox-key-push-progress` | `{ botId, phase: 'writing-env' \| 'updating-proxy-env' \| 'persisting-identity' \| 'ensuring-project' \| 'reconciling' \| 'pulling' \| 'pushing' \| 'done' \| 'failed', percent, message, at }` |

Clients render push progress from the stream regardless of who initiated the push.

## Agent Selector Re-validation

The hook keys its query on `(roomId, botId)`:

```ts
useBotPrivOSSandboxKeyStatus({ roomId, botId: selectedAgent?.botUserId });
```

Switching agents triggers a fresh fetch instead of returning the cached default-bot status. Dismissal state is keyed by `${roomId}:${botId}` so dismissing the CTA for one agent doesn't hide it after switching to another. Push polling continues every 15s, on mount, and on window focus while the chat is open.

## Mid-Session Context Handover

When the user switches agents *inside* an existing session, the new bot's
sandbox task has zero conversation history. `buildContextForStreaming`
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
again — that sandbox task missed the messages exchanged with the other agent
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

Sandbox skills call back as the bot itself. For trigger management
(`agents.triggers.*` only), `verifyBotOwnership` accepts:

1. The bot's `_createdBy` user.
2. Anyone with `owner` or `leader` role on the bot's `customFields.agentRoomId`. The bot is provisioned with `owner` on its agent room (`agents.ts:462`), so a bot using its own bearer token passes this check.
3. Server admins (`view-user-administration`).

Other endpoints that consume the same bearer token (`agents.sandbox.*`,
`agents.reply`, etc.) keep their original auth checks unchanged.

## Failure Modes

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| 401 on bot-bearer call to chat | Token rotated since last push | Press "Push bot key to PrivOS Sandbox" again |
| 401 on bot-bearer call to chat | Bot user disabled / `active: false` token | Re-provision via Agent Builder |
| 400 `bot-not-room-default` | (legacy) Bot lacked `customFields.botDefaultOfRoomId === roomId` | Now relaxed: any bot subscribed to the room is valid |
| 400 `Bot not found or not authorized` from `agents.triggers.*` | Caller is the bot itself but isn't owner/leader of the agent room | Verify the bot's subscription has `owner` role |
| `hasSandbox: false` despite room config | Room's `privosSandbox.url` not set / not enabled | Configure via room settings |
| Mid-session agent switch produces blank-context replies | Sandbox task created from scratch but only got the latest turn | Fixed — `buildContextForStreaming` now detects `agentChanged` and force-pushes full history |

## Related Docs

- [Architecture](./architecture.md) — high-level system map
- [Agent Rooms](./agent-rooms.md) — room provisioning, default bot custom fields
- [Trigger API Reference](./trigger-api-reference.md) — `agents.triggers.*` endpoints
- [Self-Management Skills](./self-management-skills.md) — how skills authenticate back to chat
- [Credential Vault](./credential-vault.md) — external-API secrets injected at egress; the bot key and the vault share one proxy route but never touch each other's rows
