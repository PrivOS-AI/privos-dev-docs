# PrivOS MCP Tools — Bot

PrivOS supports two independent bot contracts: an installation-owned agent bot selected only by
trusted Hub lifecycle state, and legacy token-selected messaging where the caller supplies a bot
credential.

## Installation-owned agent bot

One active schema-v3 parent Library Runtime installation may own one dedicated agent bot. Child
Room bindings reference their parent; they never accept or snapshot a bot selector.

### Declaring and creating the bot

Apps no longer create the bot themselves — there is no creation tool or scope. Instead:

1. **The manifest declares the identity.** The app's `privos-app.json` carries an `agentBot`
   block: `{ "agentBot": { "name": "...", "slug": "..." } }`. The `slug` becomes the bot's
   username (`mcp-app-library-provisioning-v3.ts` maps `slug` → `username`; name 2–200 chars,
   slug must match the bot-username pattern). Apps cannot invent an identity at runtime.
2. **A workspace administrator authorizes creation** via
   `POST /api/v1/mcp-apps.bot-agent.create` with `{ parentInstallationId }` (requires the
   `manage-oauth-apps` permission and a same-origin Marketplace mutation). Creation also
   requires available bot license capacity.
3. **The bot's `BotTokens` row is provisioned at creation** (`mcp-installation-agent-bot.ts`),
   so hub-side bot-key pushes and sandbox chat authenticate without a manual first push.

An app that declares no `agentBot` gets `BOT_AGENT_NOT_DECLARED` from the admin route. The
association survives an in-place generation upgrade of the same installation; a new manifest
declaration is required to move the identity.

### Issuing and receiving the credential

Creating the bot does **not** deliver a usable credential — that is a separate, admin-driven step.
The credential is the pair the Hub authenticates REST calls from: `x-user-id` (the bot user id)
plus `x-auth-token` (the secret). A backend calls ordinary Hub REST as its bot with this pair; the
frontend never touches it.

**Declaring where the credential lands (required for automatic delivery).** For the Hub to deliver
the credential to your app automatically, the manifest MUST declare **both** reserved env keys — one
without the other is refused, because the pair authenticates together:

```jsonc
"env": [
  { "key": "PRIVOS_AGENT_BOT_CREDENTIAL", "required": false, "secret": true },
  { "key": "PRIVOS_AGENT_BOT_USER_ID",    "required": false, "secret": false }
]
```

Declaring `agentBot` alone is **not** enough. An app that declares the bot but omits these keys is
handled correctly by the Hub, but falls into the show-once fallback below.

**Issuing.** A workspace admin issues (and later re-issues) the credential from
Admin > Apps > {app} > Settings — never the app itself. Re-issue is an atomic rotation: the previous
credential dies immediately and exactly one live replacement is minted.

**Delivery** depends on the installation's runtime mode:

| Mode | How it arrives | When the running app sees it |
|------|----------------|------------------------------|
| Managed / marketplace (`PRIVOS_MANAGED_RUNTIME`) | Written into the app's encrypted env config | As the two env vars, on container **(re)start** — after the admin clicks **Apply environment**. There is no live config-fetch. |
| Standalone / relay (self-host, dev) | Pushed as an ES256-signed control notification over the Relay WebSocket | **In-process, hot** — the SDK persists it to the standalone identity file and adopts it live, **no restart**. Offline at issue time ⇒ redelivered on next reconnect. |

Because of the relay path, `process.env.PRIVOS_AGENT_BOT_CREDENTIAL` can be empty while the app
still holds a working credential. Read it through the SDK (`readAgentBotCredential()` /
`createAgentBotHubClient`, see [auth-and-rest-integration.md](../auth-and-rest-integration.md)),
which checks the env pair first and the hot-adopted value second — never `process.env` directly.

**Fallback (show-once).** If the manifest does not declare both keys — or the Hub has no secret-store
encryption key — the Hub cannot deliver automatically. It then shows the credential to the admin
**exactly once** at issue time with the note *"This app does not declare a field to receive
credentials automatically. Copy this value now."* The admin pastes it into the app's own secret
config by hand. "A credential is live" in Settings means one exists server-side, not that the app
received it.

### Room tools

| Tool | Scope | Arguments | Result |
|------|-------|-----------|--------|
| `mcpapp.bot.joinCurrentRoom` | `bot:room:join` | none | Safe bot identity, exact authorized `roomId`, and `joined` |
| `mcpapp.bot.getCurrentRoomIdentity` | `bot:identity:read` | none | Safe bot identity, exact authorized `roomId`, and `membershipStatus: "member"` |

The Room tools are separately approved, interactive actions. Their schemas accept no `roomId`,
`botId`, or `botToken`; the Hub resolves the exact active child binding and verifies the current
user's Room access. Joining uses ordinary Room invitation authority and creates an ordinary
membership. Identity is returned only while that membership exists.

Creation does not mint, return, store, or push a bot secret. Sandbox key provisioning remains a
separate operation under `sandbox:botkey:push`. Successful uninstall releases the installation
association without deleting the reusable bot or changing its Room memberships.

Stable denial codes include `BOT_AGENT_NOT_DECLARED`, `BOT_AGENT_CREATE_DENIED`,
`BOT_AGENT_USERNAME_EXISTS`, `BOT_AGENT_LICENSE_LIMIT_REACHED`,
`BOT_AGENT_CREATION_IN_PROGRESS`, `BOT_AGENT_CONFIGURATION_INVALID`,
`MCP_ROOM_SCOPE_DENIED`, and `BOT_NOT_MEMBER_OF_AUTHORIZED_ROOM`.

### SDK example

```typescript
// The bot already exists: declared in the manifest, created by a workspace admin.
// No Room or bot selector: the Hub supplies the exact authorized Room binding.
await app.callServerTool({ name: 'mcpapp.bot.joinCurrentRoom', arguments: {} });
const identity = await app.callServerTool({
  name: 'mcpapp.bot.getCurrentRoomIdentity',
  arguments: {},
});
```

The [PrivOS demo MCP app](https://github.com/PrivOS-AI/privos-mcp-app-demo/blob/main/src/ui/agent-bot-panel.tsx)
contains an interactive reference panel for this flow.

## Legacy token-selected messaging

Apps can use the legacy message tools by passing a bot token in tool arguments. The bot token is generated by the bot owner via `/api/v1/bot.tokens.generate` (see [Bot REST API](../../bot-api.md) if available) and then provided to the app — typically stored via `mcpapp.app.setLocalData` or entered in the app's settings UI.

### Auth model

```
[App tool call]
   ↓ MCP scope check: app declared `bot:message:send`?    ← install-time consent
   ↓ Token validation: BotTokenService.validateToken()    ← runtime identity
   ↓ Bot is a member of the room?                         ← service guard
[Send message]
```

Two layers of access control:

- **Scope** (`bot:message:send`) — admin grants this when installing the app, signaling consent that the app may send messages on behalf of bots.
- **Bot token** — runtime credential proving which bot is acting. Without a valid token, no scope is enough.

The bot token itself is **never logged**. Only `appId` and `botUserId` appear in audit logs.

### DM safety

`mcpapp.bot.sendDirectMessage` enforces a "shared room" guard: the bot must already share at least one room with the target user. This prevents apps from spamming arbitrary users by harvesting `userId`s.

### Tools

| Tool | Scope | Purpose |
|------|-------|---------|
| `mcpapp.bot.getMe` | — | Verify a bot token, return `{ _id, username, name }` |
| `mcpapp.bot.sendMessage` | `bot:message:send` | Send text to a room (with optional reply, inline keyboard) |
| `mcpapp.bot.sendDirectMessage` | `bot:message:send` | Send DM to a user (resolves DM room automatically) |
| `mcpapp.bot.sendAttachment` | `bot:message:send` | Send photo/video/audio/document/voice to a room |

---

### `mcpapp.bot.getMe`

Validate a bot token and return the bot identity. Useful for displaying the active bot in the app UI.

### Arguments

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `botToken` | string | yes | Bot token (`privos_...`) |

### Response

```json
{
  "_id": "bot_abc123",
  "username": "support-bot",
  "name": "Support Bot"
}
```

---

### `mcpapp.bot.sendMessage`

Send a text message to a room as the bot.

### Arguments

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `botToken` | string | yes | Bot token |
| `roomId` | string | yes | Target room — bot must be a member |
| `text` | string | yes | Message text |
| `replyToMessageId` | string | no | If set, sent as a reply (quote attachment) |
| `inlineKeyboard` | object | no | Inline keyboard buttons (Telegram-style) |

### Response

```json
{ "messageId": "msg_xyz", "roomId": "room_abc", "botId": "bot_abc123" }
```

### Errors

- `botToken is required` — missing token
- `Invalid or expired bot token` — token validation failed
- `Bot is not a member of this room` — bot has no subscription
- `Failed to send message` — service rejected (quota, room locked, etc.)

---

### `mcpapp.bot.sendDirectMessage`

Send a DM to a user. The DM room between bot and user is created on demand if it doesn't exist.

### Arguments

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `botToken` | string | yes | Bot token |
| `toUserId` | string | yes | Target user — must share at least one room with the bot |
| `text` | string | yes | Message text |
| `replyToMessageId` | string | no | Reply target |
| `inlineKeyboard` | object | no | Inline keyboard buttons |

### Response

```json
{ "messageId": "msg_xyz", "roomId": "dm_bot_user", "botId": "bot_abc123" }
```

The returned `roomId` is the DM room and may be reused for follow-up messages without going through this tool again — call `mcpapp.bot.sendMessage` with that `roomId` instead.

### Errors

- `Bot has no shared room with target user` — DM safety guard
- `Bot cannot DM itself`
- `Target user not found`

---

### `mcpapp.bot.sendAttachment`

Send a media or file attachment. Three source variants are supported — pick the one that fits your data:

| `source` field | When to use | Limit |
|---|---|---|
| `fileUrl` | The file is already hosted somewhere reachable from the server | none (server streams it) |
| `base64Data` | The app generated bytes in memory (e.g. exported a chart, fetched from API) | 8 MB binary |
| `fileId` | The file already lives in PrivOS Uploads. **For files > 8 MB**, upload via the bridge with `uploadOnly: true` (see [Uploading files to a room](../developer-guide.md#uploading-files-to-a-room-roomsupload)) and pass the resulting `file._id` here — the bot will be the only message author. Plain `/api/v1/rooms.upload` (no `uploadOnly`) also posts a user message, leaving 2 messages in the room. | none |

The bot is the **author** of both the message and the upload when `base64Data` or `fileUrl` is used — exactly one message is created, owned by the bot.

### Arguments

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `botToken` | string | yes | Bot token |
| `roomId` | string | one of | Target room — bot must be a member |
| `toUserId` | string | one of | Target user for DM. DM room is auto-resolved (and created if missing). Mutually exclusive with `roomId`. |
| `type` | enum | yes | `photo` \| `video` \| `audio` \| `document` \| `voice` |
| `source.fileId` | string | one of | Existing PrivOS upload ID |
| `source.fileUrl` | string | one of | External URL — server fetches it |
| `source.base64Data` | string | one of | Base64-encoded bytes. The `data:<mime>;base64,` prefix is stripped automatically. Max 8 MB binary (≈ 10.7 MB encoded) |
| `source.filename` | string | required with `base64Data` | Filename used when uploading inline bytes |
| `source.mimeType` | string | recommended with `base64Data` | Defaults to `application/octet-stream` if omitted |
| `filename` | string | no | Override filename (otherwise inferred from source) |
| `caption` | string | no | Photo caption (only used when `type === 'photo'`; falls back to caption if `text` is empty) |
| `text` | string | no | Message text sent alongside the attachment (works for all types — produces a single message with both text and file, like dragging a file into the chat with a caption) |
| `replyToMessageId` | string | no | Reply target |

### Response

```json
{ "messageId": "msg_xyz", "roomId": "room_abc", "botId": "bot_abc123" }
```

### Type rules

| Type | Allowed mime/extensions |
|------|-------------------------|
| `photo` | `image/*` (EXIF stripped if `Message_Attachments_Strip_Exif` is on) |
| `audio` | `.mp3`, `.m4a` only |
| `voice` | `.ogg`, `.oga`, `.opus`, `.mp3`, `.webm` |
| `video` | `video/*` |
| `document` | most `application/*` and `text/*` types — max 50 MB |

### Errors

- `roomId or toUserId is required`
- `Provide either roomId or toUserId, not both`
- `Bot is not a member of this room` — when `roomId` is used and bot has no subscription
- `Target user not found` — `toUserId` does not exist
- `source must contain fileUrl, fileId, or base64Data`
- `File not found` — `fileId` doesn't exist in `Uploads`
- `Failed to fetch fileUrl: <status>` — external URL unreachable
- `base64Data decoded to zero bytes — check encoding` — empty/malformed base64
- `base64Data exceeds 8388608 bytes; use fileUrl or upload via REST first` — payload too large
- `Failed to send attachment` — service rejected (mime type, size, room access)

---

### Example (SDK)

```typescript
const app = usePrivosApp();
const botToken = await app.callServerTool({
  name: 'mcpapp.app.getLocalData',
  arguments: { key: 'botToken' },
});

await app.callServerTool({
  name: 'mcpapp.bot.sendMessage',
  arguments: {
    botToken: botToken.value,
    roomId: ctx.roomId,
    text: 'Hello from the app!',
  },
});
```

### Token storage recommendations

- Store the token via `mcpapp.app.setLocalData` so it persists across sessions (scoped per app).
- Never expose the token in client-rendered HTML or non-bot tool responses.
- Rotate via `/api/v1/bot.tokens.invalidate` then `/v1/bot.tokens.generate` when an app loses access.
