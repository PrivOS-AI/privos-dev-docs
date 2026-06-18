# Auth Model & REST-First Integration

> **Direction:** PrivOS Hub's existing REST API (`/api/v1/*`) is the single integration
> surface for app data access. Do **not** add per-resource server tools (`privos.*` /
> `mcpapp.*`) that re-wrap functionality the REST API already exposes. Call the REST
> endpoints directly with the right credential for your context.
>
> The legacy `privos.*` / `mcpapp.*` `callServerTool` tools (see [api-reference.md](./api-reference.md))
> remain for backward compatibility but are **not** the path for new capabilities.
> Runtime-only concerns that have no REST equivalent stay as tools (see
> [Tools with no REST equivalent](#tools-with-no-rest-equivalent)).

## Why REST-first

- **No duplication.** Resource logic (permission/ACL checks, duplicate handling,
  embedding, webhooks) already lives in the REST routes. A second copy in tool handlers
  drifts and can be looser — e.g. the current `mcpapp.files.*` tools call models directly
  and skip the per-user room ACL that `file-management.files.upload` enforces.
- **Every new hub API is instantly usable** by apps — no tool needs to be written/registered.
- **Permissions are enforced by the credential**, not by a hand-maintained scope-per-tool table.

## Two auth contexts

| Context | Credential | Identity (whose permissions) | Who provisions |
|---|---|---|---|
| **Frontend** (iframe UI) | The logged-in user's session token, used **by the host** | The current user | Nothing — reuse the existing login |
| **Backend** (app's own MCP/server process) | A **bot token** (`privos_…`) in `X-Auth-Token` + `X-User-Id` | A dedicated least-privilege bot user | App owner configures it once |

### Frontend — reuse the user session (no bot token)

The app UI runs in a **deny-by-default sandbox** (`sandbox="allow-scripts"`, **no**
`allow-same-origin` — see `client/views/room/mcp-apps/use-mcp-bridge-host.ts`,
`McpAppHost.tsx`). The iframe therefore **cannot** read the user's cookie/login token.

Instead, the **host** (PrivOS parent page, same origin) holds the session and calls the
hub API as the user — it reads `Meteor.loginToken` + `X-User-Id` and attaches them as
auth headers (`client/views/mcp-apps/McpAppStandalonePage.tsx`). The sandboxed iframe
asks the host over the postMessage bridge; the host performs the REST call.

**Least-privilege at the app tier — allowlist derived from granted scopes.** A host that
proxies *any* REST call as the user would hand every installed app the user's full power.
So the bridge MUST gate each request. The declaration unit stays the existing
**`scopes`** (`mcp_apps.scopes`, `IMcpApp.scopes`) — no new per-app field. A central
static **scope → `{method, path-prefix}[]` map** (one code module) expands the app's
granted scopes into the effective allowlist (e.g. `files:write` →
`POST file-management.files.*`, `DELETE file-management.files/:id`). The host bridge
rejects any request whose `{method, path}` doesn't match an entry derived from
`app.scopes`.

- Declaration: `scopes` only — requested in the manifest, granted on `mcp_apps.scopes`
  (admin-approved); the scope→path map lives in code, not in per-app data.
- The raw `Meteor.loginToken` MUST never be forwarded into the sandboxed iframe.
- The allowlist is enforced in the host (trusted), never in the iframe (untrusted).
- **Prerequisite:** registration currently hardcodes `scopes = ALL_SCOPES`
  (`server/services/mcp-app-lifecycle-service.ts`). This must change to grant only the
  manifest-requested subset, otherwise the allowlist is meaningless (every app gets every
  path).

### Backend — app-owner-configured bot token

A headless app backend has no browser session, so the app owner provisions a **bot token**:

```
POST /api/v1/bot.tokens.generate        # mints a privos_… token for a bot user
```

The backend then calls any hub REST endpoint as that bot user:

```bash
curl -X POST https://<hub>/api/v1/file-management.files.upload \
  -H "X-Auth-Token: $PRIVOS_BOT_TOKEN" \
  -H "X-User-Id: $PRIVOS_BOT_USER_ID" \
  -F channelId=$ROOM_ID -F files=@./report.pdf
```

(Header auth is the standard session-token path — `app/api/server/ApiClass.ts`,
"Session auth (X-Auth-Token + X-User-Id)".)

## Calling the Sandbox agent (REST client-facing)

To run a PrivOS Sandbox (claude-ws) agent and get its text back, call the hub REST
proxy — the caller never needs its own Sandbox URL/API key. Two endpoints
(`app/api/server/v1/agent-privos-sandbox-proxy-endpoints.ts`), both `authRequired` and
gated on the caller having access to `roomId`:

| Endpoint | Method | Purpose |
|---|---|---|
| `agents.sandbox.upload` | POST | Upload a file, get a `tempId` to attach to a generation |
| `agents.sandbox.generate` | POST | One-shot **synchronous** agent generation → `{ text }` |
| `agents.sandbox.generate-async` | POST | Enqueue a generation → `{ attemptId, taskId }` immediately |
| `agents.sandbox.attempt-status` | GET | Poll an async generation → `{ status, text?, json? }`; add `&partial=1` to stream blocks while running |

### Call — `agents.sandbox.generate`

```bash
curl -X POST https://<hub>/api/v1/agents.sandbox.generate \
  -H "X-Auth-Token: $PRIVOS_BOT_TOKEN" \
  -H "X-User-Id: $PRIVOS_BOT_USER_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "roomId": "ROOM_ID",
    "prompt": "Summarise the attached report",
    "provider": "anthropic",          // optional — falls back to room/global default
    "model": "claude-...",            // optional
    "fileIds": ["TEMP_ID"],           // optional — from agents.sandbox.upload
    "systemContext": "..."            // optional
  }'
# → { "text": "...", "source": "room" | "global" }
```

Limits: `prompt` < 50000 chars, `systemContext` < 200000 chars.

### Response model — sync (blocking) or async + poll

**Sync — `generate`** blocks until the agent finishes and returns the full text in the
single HTTP response. The hub does all the waiting server-side
(`privos-sandbox-agent-service.ts` `syncResponseWithConfig`):

- Sandbox runs in `request_method: 'sync'` (its sync window is ~5 min).
- On a `408` (window elapsed, agent still running) the hub parses the `attemptId` and
  **polls internally** until the attempt terminates, then extracts text from its logs
  (`pollAttemptUntilDone`, capped at 15 min).
- On gateway timeouts (`502/504/524`) the hub recovers by looking up the latest attempt
  for the `taskId` and polling that.

So a sync caller's only job is to **set a generous client HTTP timeout** (the server caps
the wait at ~15 min via `POLL_MAX_MS`); the hub returns once, with the final text.

**Async + poll — `generate-async` / `attempt-status`** for callers that can't hold a long
HTTP request open (MCP app iframes behind the 10s bridge timeout, serverless, etc.):

```bash
# 1. Enqueue — returns immediately
curl -X POST https://<hub>/api/v1/agents.sandbox.generate-async \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $USER_ID" -H "Content-Type: application/json" \
  -d '{ "roomId": "ROOM_ID", "prompt": "..." }'
# → { "attemptId": "agent-attempt-…", "taskId": "...", "source": "room" }

# 2. Poll until terminal (e.g. every 2–5s)
curl "https://<hub>/api/v1/agents.sandbox.attempt-status?roomId=ROOM_ID&attemptId=agent-attempt-…" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $USER_ID"
# while running → { "status": "running" }
# when done    → { "status": "completed" | "failed" | "cancelled", "text": "...", "json": [ ... ] }
```

The hub keeps **no job state** — the attempt lives in the Sandbox (enqueued with
`request_method: 'queue'`); `attempt-status` proxies the Sandbox's status and, on a
terminal status, extracts the assistant text from the attempt logs
(`startAsyncResponseWithConfig` / `getAttemptResultWithConfig`). Both endpoints re-check
room access on every call. Note: polling returns the final text only — token-by-token
deltas remain hub-internal (see Reachability below).

On a terminal status the response also carries **`json`** — the array of structured
response blocks parsed from the attempt's `type:'json'` log lines (the same events the
hub flattens into `text`). Use `text` for the plain reply; use `json` when you need the
agent's structured output (tool calls, `result` summaries, content blocks, usage, etc.).
The array is the parsed events *after the last user turn* of the attempt. By default it
is omitted while `status` is `running`.

**Streaming blocks while running — `&partial=1`.** Pass `partial=1` on `attempt-status`
to also receive the `text`/`json` accumulated **so far** on a `running` poll, instead of
waiting for the terminal status. The attempt's logs grow as the agent works, so each poll
returns more blocks — letting a client render the response **block-by-block as it
generates** (assistant text, tool calls, tool results, result summary):

```bash
curl "https://<hub>/api/v1/agents.sandbox.attempt-status?roomId=ROOM_ID&attemptId=…&partial=1" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $USER_ID"
# running  → { "status": "running", "text": "<partial>", "json": [ <blocks so far> ] }
# terminal → { "status": "completed", "text": "...", "json": [ ... ] }
```

Notes: `partial` is **opt-in** — without it, behaviour is unchanged (`running` carries no
`text`/`json`). Each partial poll fetches the full attempt log (heavier than a bare status
check), and the partial fetch is best-effort: if it fails mid-run the poll degrades to a
plain `{ "status": "running" }`. Poll a bit faster (~1–2s) when streaming. This is still
event-level (per assistant message / tool call), **not** token-level — token deltas remain
hub-internal (see Reachability below).

### Uploading a file first

```bash
curl -X POST https://<hub>/api/v1/agents.sandbox.upload \
  -H "X-Auth-Token: $PRIVOS_BOT_TOKEN" -H "X-User-Id: $PRIVOS_BOT_USER_ID" \
  -H "Content-Type: application/json" \
  -d '{ "roomId": "ROOM_ID", "fileName": "report.pdf", "mimeType": "application/pdf", "base64": "..." }'
# → { "tempId": "...", "source": "room" | "global" }
```

Pass the returned `tempId` in `generate`'s `fileIds`.

### Reachability

- **Backend (bot token):** works — these are normal header-auth REST routes.
- **Frontend (`app.rest()`):** reachable with the **`sandbox:generate`** scope, which maps
  to `agents.sandbox.generate` / `generate-async` / `attempt-status` / `upload`
  (`server/services/mcp-rest-allowlist.ts`). In-iframe apps should use the **async + poll**
  pair — the host-bridge postMessage default times out at 10s (`PrivOSAppProvider`
  `sendRequest`), while a sandbox generation can take minutes:

  ```ts
  // 1. Enqueue (fast — fits the 10s bridge timeout)
  const { body: started } = await app.rest({
    method: 'POST',
    path: 'agents.sandbox.generate-async',
    body: { roomId, prompt: 'Summarise the report' },
  });

  // 2. Poll until terminal
  let result;
  do {
    await new Promise((r) => setTimeout(r, 3000));
    ({ body: result } = await app.rest({
      method: 'GET',
      path: 'agents.sandbox.attempt-status',
      query: { roomId, attemptId: started.attemptId },
    }));
  } while (result.status === 'running');
  // result.text — the agent's plain reply
  // result.json — structured response blocks (tool calls, result summary, content blocks)
  ```

  The synchronous `agents.sandbox.generate` stays available for backend bot-token callers
  that can hold the request open.
- **Streaming** (token-by-token `output:json`, `question:ask`) exists only on the hub's
  *internal* proxy (`app/agent-chat/server/privosSandboxProxy.ts`) for the hub's own
  agent-chat UI — it is **not** part of the client-facing REST surface.

## Provisioning the room's Sandbox (bot-key push)

Before a room can run agents, its Sandbox **project** must be provisioned and the room
bot's key pushed. The push uploads the room's config templates to MinIO, registers the
key on the Sandbox, and warms a VM — i.e. "init sandbox" and "push bot key" are the same
operation. Two REST routes (`app/api/server/v1/agent-privos-sandbox-bot-key.ts`):

| Path | Method | Purpose |
| --- | --- | --- |
| `agents.sandbox.botKeyStatus` | GET | Read push status → `{ pushed, hasBot, hasSandbox, canPush, status? }` |
| `agents.sandbox.pushBotKey` | POST | Provision project + push/refresh the bot key |

### Reachability

- **Frontend (`app.rest()`):** reachable with the **`sandbox:botkey:push`** scope, which
  maps to both routes (`server/services/mcp-rest-allowlist.ts`).
- **Server-side gate is unchanged by the scope.** `pushBotKey` still enforces
  `authorizePushBotKey` — the caller must hold **`edit-room`** AND be the bot's owner OR
  hold **`edit-bot`**. The call runs as the end-user session (forwarded `X-Auth-Token`),
  never a bot token, so the scope only widens reachable paths — it never changes who may push.

```ts
// 1. Check whether the room is already provisioned
const { body: status } = await app.rest({
  method: 'GET',
  path: 'agents.sandbox.botKeyStatus',
  query: { roomId },
});

// 2. Provision + push if allowed (status.canPush === true for room admins)
if (!status.pushed && status.canPush) {
  await app.rest({ method: 'POST', path: 'agents.sandbox.pushBotKey', body: { roomId } });
}
```

```bash
curl -X POST https://<hub>/api/v1/agents.sandbox.pushBotKey \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $USER_ID" \
  -H "Content-Type: application/json" \
  -d '{ "roomId": "ROOM_ID" }'
# → { "success": true, "privosSandboxId": "...", "pushedAt": "..." }
```

## Selecting room skills (sync + remove)

Per-room skill selection is a single **sync** endpoint that takes the **full desired set**
of `componentIds`. Adding a skill = include its id; removing one = re-sync without it.
There is no separate delete endpoint — sync is the add/remove mechanism. Both routes
(`app/api/server/v1/rooms.ts`):

| Path | Method | Purpose |
| --- | --- | --- |
| `rooms.listPrivOSSandboxSkills` | POST | List the skills available to the room → `{ skills: [{ id, name, description? }] }` |
| `rooms.syncPrivOSSandboxSkills` | POST | Set the room's enabled skills to exactly `componentIds` |

### Reachability

- **Frontend (`app.rest()`):** reachable with the **`sandbox:skills:use`** scope.
- **Server-side gate:** both routes enforce **`edit-room`** (room admin), independent of the
  scope. The selected skills sync into the room project's `.privos/skills/` and are picked
  up on the next agent generation.

```ts
// List options
const { body: avail } = await app.rest({
  method: 'POST', path: 'rooms.listPrivOSSandboxSkills',
  body: { rid: roomId, useGlobal: true },
});

// Enable exactly these (omit an id to remove it)
await app.rest({
  method: 'POST', path: 'rooms.syncPrivOSSandboxSkills',
  body: { rid: roomId, componentIds: ['skill-a', 'skill-b'] },
});
```

## Security — bot token leakage & privilege escalation

A bot/session token is a **bearer credential**: whoever holds it acts as that principal,
with its full permissions, from anywhere. Mitigate blast radius:

- **Never use a high-privilege / admin user's key** for an app. One leak = full takeover.
- **Dedicated least-privilege bot user per app** — grant only the rooms/permissions the app
  needs. One app's compromise must not reach another's data.
- **Short-lived + rotatable + revocable** tokens (the `BotTokens` store supports expiry);
  revoke immediately on suspected leak.
- **Never log** `Authorization` / `X-Auth-Token` headers; store the token in a secret
  manager, not in code or app config dumps. TLS only.
- **Frontend**: enforce the per-app allowlist host-side; never leak the user login token
  into the iframe.

## Legacy tool → REST endpoint mapping

For new work, replace these `callServerTool` calls with the REST endpoint. All file routes
verified in `app/api/server/v1/fileManagement.ts`; others in the named route file.

| Legacy tool | REST endpoint |
|---|---|
| `privos.files.getByChannel` | `GET file-management.files.channel/:channelId` |
| `privos.files.get` | `GET file-management.files/:fileId` |
| `privos.files.search` | `GET file-management.files.search/:channelId` |
| `privos.files.update` | `POST file-management.files/:fileId/rename` (or `/update-content`) |
| `privos.files.delete` | `DELETE file-management.files/:fileId` |
| *(file upload — was never a tool)* | `POST file-management.files.upload` |
| *(large file upload)* | `POST file-management.files.upload-chunked-init` → `…upload-chunk` → `…upload-chunked-complete` / `…-cancel` |
| `privos.folders.*` | `file-management.folders.*` (`channel/:channelId`, `:folderId`, create/rename/delete) |
| `privos.lists.*` | `lists.*` (`lists.create`, `lists.list`, `lists.info`, `lists.update`, items via `lists.*`/`items.*`) |
| `privos.lists.addField` / `removeField` | `lists.fields.create` / `lists.fields.delete` (or `lists.addField` / `lists.removeField`) |
| `privos.messages.send` | `POST chat.sendMessage` (or `chat.postMessage`) |
| `privos.messages.getRecent` | `GET channels.messages` / `channels.history` |
| `privos.rooms.get` | `GET channels.info` |
| `privos.rooms.getMembers` | `GET channels.members` |
| `privos.users.get` / `getByIds` | `GET users.info` / `users.list` |
| `privos.bot.sendMessage` / `sendAttachment` | `POST chat.sendMessage` with a bot token (identity = bot user) |

> File **archive extraction** (unzip into the folder tree) has **no** REST endpoint today.
> If needed, add it as a single new REST route (e.g. `file-management.files.extract`) —
> not as a tool — so it stays on the one integration surface.

## Tools with no REST equivalent

These are MCP-runtime / sandbox concerns, not hub resources — keep using them via the
bridge/SDK:

- `privos.context.get` — current room/user context for the iframe session.
- `privos.app.getLocalData` / `setLocalData` / `deleteLocalData` / `clearLocalData` — the
  app's own sandboxed key-value store.
- `privos.db.*` — the app-private collection store (app's own data, not a hub resource).
- `privos.bot.getMe` — runtime bot-token identity check.

## Implemented approach (allowlist)

- **Declaration unit = `scopes`** (no new field). Manifest requests scopes (validated by
  `mcp-manifest-fetcher.ts`) → granted on `mcp_apps.scopes` at registration
  (`mcp-app-lifecycle-service.ts`). Apps that declare nothing get **no** scopes (fail-safe);
  existing apps keep their prior grant (refresh does not re-narrow).
- **Central `scope → {method, path-prefix}[]` map**: `server/services/mcp-rest-allowlist.ts`
  (`isRequestAllowed(method, path, scopes)`) expands granted scopes into the REST allowlist.
- **Enforcement is server-side (authoritative):** route `POST /api/v1/mcp-apps.rest-call`
  runs the call as the user and rejects out-of-allowlist requests (403). The host bridge
  forwards to it (`host/rest.request`); the iframe never sees the token.
- **Multipart upload** can't tunnel through the JSON proxy, so `host/file.upload` posts
  directly to `file-management.files.upload` as the user; the host gates it on `files:write`
  (the iframe reaches REST only via the trusted host bridge).

### SDK usage (`@privos/app-react`)

```ts
const app = usePrivOSApp();

// Read files in the room (needs files:read)
const res = await app.rest({ method: 'GET', path: `file-management.files.channel/${roomId}` });

// Send a message (needs messages:send)
await app.rest({ method: 'POST', path: 'chat.sendMessage', body: { message: { rid: roomId, msg: 'hi' } } });

// Upload a file (needs files:write)
await app.uploadFile({ channelId: roomId, fileName: 'report.pdf', base64Data, mimeType: 'application/pdf' });
```

Manifest declares requested scopes:

```json
{ "name": "my-app", "version": "1.0.0", "scopes": ["files:read", "files:write", "messages:send"] }
```

### Deferred (YAGNI)
Per-path granularity beyond scopes, or per-room differentiation — add `restAllowlist` to
`IMcpApp` / `mcp_app_installations` only if a real need appears.

## Open questions

- Deprecation timeline for the resource-wrapping `privos.*` / `mcpapp.*` tools — keep as
  thin REST shims, or remove after consumers migrate? (Audit current callers first.)
