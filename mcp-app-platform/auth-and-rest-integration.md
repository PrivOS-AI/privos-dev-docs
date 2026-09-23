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
| **Backend** (app's own MCP/server process) | A bot credential in `X-Auth-Token` + `X-User-Id` | A dedicated least-privilege bot user | Legacy: app owner generates a token once. Schema-v3 (recommended): the installation-owned agent bot, Hub-provisioned and delivered — see below |

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

This is the **legacy, manually-provisioned** path: the app owner generates the token and stores it
in the app's config by hand. It still works, but a schema-v3 Library Runtime app should prefer the
installation-owned agent bot below, which the Hub provisions and delivers for you.

### Backend — installation-owned agent bot (schema-v3, recommended)

Instead of a hand-generated token, the app declares a bot in its manifest and the Hub mints,
delivers, and rotates the credential. The backend never generates or stores the secret itself.

**1. Declare the bot and where the credential lands.** The manifest needs the `agentBot` block
**and both** reserved env keys (one without the other is refused — the pair authenticates together):

```jsonc
{
  "agentBot": { "name": "My App Bot", "slug": "my-app-bot" },
  "env": [
    { "key": "PRIVOS_AGENT_BOT_CREDENTIAL", "required": false, "secret": true },
    { "key": "PRIVOS_AGENT_BOT_USER_ID",    "required": false, "secret": false }
  ]
}
```

Declaring `agentBot` alone is not enough: without the env keys the Hub falls back to a show-once
value the admin must copy by hand. See
[apis/tools-bot.md § Issuing and receiving the credential](apis/tools-bot.md#issuing-and-receiving-the-credential)
for the full delivery contract (managed = env at restart; relay = hot push, no restart).

**2. Call Hub REST as the bot via the SDK.** `createAgentBotHubClient` re-reads the credential on
every call (so a re-issue is picked up hot) and attaches the `x-user-id` / `x-auth-token` pair. It
reads the env pair first, then the value the Hub hot-adopted over the relay channel — so
`process.env` being empty is normal and correct. **Never read `process.env` directly.**

```ts
import { createAgentBotHubClient } from '@privos_ai/app-server';
import { resolveHubOrigin } from './resolve-hub-origin'; // mode-aware Hub origin lookup

const hub = createAgentBotHubClient({ resolveHubOrigin });

const res = await hub.authorizedFetch('/api/v1/mcp-apps.tool-call', {
  method: 'POST',
  requiredScope: 'db:write',        // documentation-only at the client; the Hub enforces server-side
  retryMode: 'never',               // 'never' for non-idempotent writes; 'safe-methods' | 'idempotent' otherwise
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ mcpAppId, toolName, arguments: args, roomId }),
});
if (!res.ok) {
  // 401 = credential wrong/rotated (distinct from absent); other 4xx/5xx = Hub refusal.
  throw new Error(`Hub REST as bot failed: HTTP ${res.status}`);
}
```

Mode-specific factories exist when you would rather not write `resolveHubOrigin`:
`createAgentBotHubClientFromWorkloadIdentity(client)` (managed) and
`createAgentBotHubClientFromHubOrigin(origin)` (standalone). For the raw pair or a diagnostic state,
use `readAgentBotCredential()` (`{ botUserId, token } | null`) and `getAgentBotCredentialState()`
(`'absent' | 'live' | 'rejected'`). Errors: `AgentBotCredentialAbsentError` (nothing delivered yet —
degrade, don't crash) and `AgentBotHubUnreachableError` (origin unresolved / Hub unreachable — not a
401).

The bot only reaches endpoints its granted scopes allow (Hub maps scope → REST path via the
allowlist); `requiredScope` at the client is documentation, not the authority.

## Calling the Sandbox agent (REST client-facing)

To run a PrivOS Sandbox (claude-ws) agent and get its text back, call the hub REST
proxy — the caller never needs its own Sandbox URL/API key. Two endpoints
(`app/api/server/v1/agent-privos-sandbox-proxy-endpoints.ts`), both `authRequired` and
gated on the caller having access to `roomId`:

| Endpoint | Method | Purpose |
|---|---|---|
| `agents.sandbox.upload` | POST | Upload a file, get a `tempId` to attach to a generation |
| `agents.sandbox.generate` | POST | One-shot **synchronous** agent generation → `{ text }` |
| `agents.sandbox.generate-async` | POST | Enqueue a generation → `{ attemptId, taskId }` immediately. Accepts an optional caller-stable `operationId` — see [Idempotent dispatch](#idempotent-dispatch-with-operationid) |
| `agents.sandbox.attempt-status` | GET | Poll an async generation → `{ status, text? }` |
| `agents.sandbox.attempt-observation` | GET | Full attempt state (phase, pending question, output) for an `operationId`-dispatched attempt — see [below](#observing-cancelling-and-reading-evidence) |
| `agents.sandbox.attempt-cancel` | POST | Cancel a running `operationId`-dispatched attempt |
| `agents.sandbox.attempt-evidence` | GET | Per-call LLM provenance for an `operationId`-dispatched attempt |

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
# when done    → { "status": "completed" | "failed" | "cancelled", "text": "..." }
```

Both `roomId` and `attemptId` are **required** query params on every `attempt-status`
poll (as they are on `attempt-observation` / `attempt-cancel` / `attempt-evidence`, and in
the request body of `generate` / `generate-async` / `upload` / `answer`). Omitting or
mistyping a required string param fails with HTTP 400
`{ "success": false, "error": "<param> is required and must be a string" }` — the offending
field is named (no opaque `Match error: Expected string, got undefined`).

The hub keeps **no job state** — the attempt lives in the Sandbox (enqueued with
`request_method: 'queue'`); `attempt-status` proxies the Sandbox's status and, on a
terminal status, extracts the assistant text from the attempt logs
(`startAsyncResponseWithConfig` / `getAttemptResultWithConfig`). Both endpoints re-check
room access on every call. Note: polling returns the final text only — token-by-token
deltas remain hub-internal (see Reachability below).

> Structured response blocks (tool calls, content blocks) are exposed on the **native AI
> Chat** path (`ai-messages.*`, scope `sandbox:ai-chat`), not on this Sandbox proxy.

### Uploading a file first

```bash
curl -X POST https://<hub>/api/v1/agents.sandbox.upload \
  -H "X-Auth-Token: $PRIVOS_BOT_TOKEN" -H "X-User-Id: $PRIVOS_BOT_USER_ID" \
  -H "Content-Type: application/json" \
  -d '{ "roomId": "ROOM_ID", "fileName": "report.pdf", "mimeType": "application/pdf", "base64": "..." }'
# → { "tempId": "...", "source": "room" | "global" }
```

Pass the returned `tempId` in `generate`'s `fileIds`.

### Idempotent dispatch with `operationId`

`generate-async` accepts an optional `operationId` (string, ≤256 chars). Pass one raw,
caller-chosen id per logical operation (e.g. a UI action's own idempotency key):

```json
{ "roomId": "ROOM_ID", "prompt": "...", "operationId": "my-caller-chosen-id" }
```

The hub derives a stable attempt identity from `{ operationId, callerId }` plus the
resolved room/executor/project/workspace/task — so a retried call with the **same**
`operationId` and the **same** request converges on the **same attempt** instead of
starting a duplicate. The response gains `adopted` / `replayed` booleans (only present
when `operationId` was supplied) so the caller can tell a fresh dispatch from one that
reused an existing attempt. If the same `operationId` is replayed with a **different**
request (different prompt, model, room, etc.), the call fails closed with `operationId
is already bound to a different Sandbox request` rather than silently running the new
request or silently returning the old result.

Passing `operationId` is also the prerequisite for the observation/cancel/evidence
endpoints below — an attempt dispatched without one has no stable identity, and those
three endpoints will return a "not found" style failure for it.

### Observing, cancelling, and reading evidence

For an attempt dispatched with `operationId`, three endpoints give an app fuller control
than plain `attempt-status` polling. All three re-check room access and re-verify the
caller is the original dispatcher (`callerId`) on every call.

**`agents.sandbox.attempt-observation`** (`GET`, query `roomId`, `attemptId`) — full attempt
state:

```jsonc
{
  "attemptId": "...", "taskId": "...", "projectId": "...", "workspaceId": "...",
  "status": "running" | "completed" | "failed" | "cancelled" | "unknown",
  "phase": "...",                       // Sandbox-reported phase string, e.g. waiting-for-user
  "createdAt": 0, "startedAt": 0, "updatedAt": 0, "completedAt": 0,
  "terminalCause": { "code": "...", "message": "..." } | null,
  "pendingQuestion": { "toolUseId": "...", "questions": [/* ... */], "timestamp": 0 } | null,
  "output": [/* ... */], "outputTruncated": false,
  "questionBridge": { "toolUseId": "...", "threadMessageId": "...", "postedMessageId": "...", "status": "..." } | null,
  "source": "room" | "global"
}
```

A non-null `pendingQuestion` (typically with `phase: "waiting-for-user"`) means the
attempt is blocked on `AskUserQuestion`; answer it with the backend-only
`agents.sandbox.answer` route (`{ roomId, attemptId, toolUseId, answers }`) — not part of
the `sandbox:generate` frontend allowlist, so only a bot-token/header-auth caller can
answer directly. Approval-class questions additionally require the answering user to
hold the room `owner` role.

**`agents.sandbox.attempt-cancel`** (`POST`, body `{ roomId, attemptId }`) → `{ status,
cancelled, ...worker fields, source }`. `status` is worker-authoritative — cancelling an
attempt that has already reached `completed`/`failed` returns that real terminal status
rather than rewriting it to `cancelled`.

**`agents.sandbox.attempt-evidence`** (`GET`, query `roomId`, `attemptId`) → `{ attemptId,
calls: [...], source }` — the per-LLM-call provenance (model/provider/tokens) recorded
for the attempt, for cost/audit purposes.

### Reachability

- **Backend (bot token):** works — these are normal header-auth REST routes.
- **Frontend (`app.rest()`):** reachable with the **`sandbox:generate`** scope, which maps
  to `agents.sandbox.generate` / `generate-async` / `attempt-status` /
  `attempt-observation` / `attempt-cancel` / `attempt-evidence` / `upload`
  (`server/services/mcp-rest-allowlist.ts`). In-iframe apps should use the **async + poll**
  pair — the host-bridge postMessage default times out at 10s (`PrivosAppProvider`
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
| `agents.sandbox.botKeyStatus` | GET | Read push status → `{ pushed, reason?, hasBot, hasSandbox, canPush, canAutoPush, status?, pushedAt?, needsForceOverwrite?, errorMessage? }`. `reason` explains a `pushed: false` (`no-record` \| `last-push-failed` \| `hash-mismatch` \| `config-missing` \| `sandbox-state-lost` \| `sandbox-key-stale`); `canAutoPush` reports that the sandbox is configured, so the server-side automatic repair can run for any member independent of the caller's own push permissions |
| `agents.sandbox.pushBotKey` | POST | Provision project + push/refresh the bot key. Body: `roomId`, optional `botId`, `auto` (marks an unattended push — the server is the sole initiator of automatic repair and bounds retries on real refusals only), `forceOverwritePersona`, `bootstrapPush` |

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

## Waking the room's VM (without re-pushing the bot key)

Once a room is provisioned, its VM is **idle-stopped** after inactivity. Pushing the bot
key already establishes the VM in the background, but re-pushing the key just to wake an
idle room is wasteful (key churn). Use **wake** to establish/wake the room's VM directly,
and **vmState** to poll the result. Establishment is silent and idempotent — concurrent
wakes (and a first chat message) coalesce into one establishment, so a duplicate wake is
ignored until the VM is up. After a successful establish the room is usable on the board
**without sending a chat message**. Two REST routes
(`app/api/server/v1/agent-privos-sandbox-bot-key.ts`):

| Path | Method | Purpose |
| --- | --- | --- |
| `agents.sandbox.wake` | POST | Wake/establish the room's VM (create + start + register on the board). Returns `{ accepted: true }` immediately |
| `agents.sandbox.vmState` | GET | Poll the VM state → `{ vmState, errorMessage? }` where `vmState ∈ running \| starting \| stopped \| error \| not_found \| unknown` |

### Reachability

- **Frontend (`app.rest()`):** both routes reachable with the **`sandbox:wake`** scope
  (`server/services/mcp-rest-allowlist.ts`).
- **Server-side gate:** `wake` enforces the **same** `authorizePushBotKey` check as
  `pushBotKey` (caller holds **`edit-room`** AND is the bot's owner OR holds **`edit-bot`**),
  runs as the end-user session, and **never writes/rotates the bot key**. `vmState` is a
  pure read gated by room access (no side effect — it never wakes the VM).

```ts
// 1. Wake / establish the room's VM (fire-and-forget; returns immediately)
await app.rest({ method: 'POST', path: 'agents.sandbox.wake', body: { roomId } });

// 2. Poll until running (vmState is a pure read — it does not wake the VM)
let vmState = 'starting';
while (vmState === 'starting' || vmState === 'stopped' || vmState === 'unknown') {
  await new Promise((r) => setTimeout(r, 2000));
  const { body } = await app.rest({
    method: 'GET', path: 'agents.sandbox.vmState', query: { roomId },
  });
  vmState = body.vmState; // 'running' = ready; 'error' = body.errorMessage explains why
  if (vmState === 'running' || vmState === 'error' || vmState === 'not_found') break;
}
```

```bash
curl -X POST https://<hub>/api/v1/agents.sandbox.wake \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $USER_ID" \
  -H "Content-Type: application/json" -d '{ "roomId": "ROOM_ID" }'
# → { "accepted": true }

curl "https://<hub>/api/v1/agents.sandbox.vmState?roomId=ROOM_ID" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $USER_ID"
# → { "vmState": "running" }
```

## Selecting room skills & agent sets

Per-room plugin selection covers standalone skills (`componentIds`) and agent sets
(`agentSetIds`). Three room routes (`app/api/server/v1/rooms.ts`):

| Path | Method | Purpose |
| --- | --- | --- |
| `rooms.listPrivOSSandboxSkills` | POST | Workspace **catalog** annotated per room → `{ skills: [{ id, name, description?, type?, selected }] }`. `type` is `'skill' \| 'agent_set'`; `selected` reflects the room's stored selection |
| `rooms.syncPrivOSSandboxSkills` | POST | Enable/disable for the room — replace mode or merge mode (below) |
| `rooms.getPrivOSSandboxSelection` | GET | Read the room's selection: hub-stored lists + the sandbox's effective (installed) selection → `{ source, stored: { skills, agentSets }, sandbox: { selectedComponents, selectedAgentSets } \| null, projectIds }` |

### Replace vs merge on `rooms.syncPrivOSSandboxSkills`

The two modes are mutually exclusive in one request:

- **Replace:** `componentIds?` / `agentSetIds?` — each supplied array is the COMPLETE
  desired selection for its own kind. **Omitted** leaves that kind untouched, an explicit
  **`[]` clears it** — never default a kind to `[]` on a one-kind sync.
- **Merge:** `addComponentIds?` / `removeComponentIds?` / `addAgentSetIds?` /
  `removeAgentSetIds?` — applied on top of the room's stored selection, so an app can
  enable its own agent set without knowing (or clobbering) the rest of the room's
  selection. `remove` wins over `add` for the same id.

The response reports `{ source, succeeded, failed, applied: { componentIds, agentSetIds } }`
— `applied` is the complete selection that was synced.

### Reachability

- **Frontend (`app.rest()`):** all three routes reachable with the **`sandbox:skills:use`**
  scope.
- **Server-side gate:** the sync/list routes enforce **`edit-room`** (room admin); the
  selection read enforces room access. The selected skills sync into the room project's
  `.privos/skills/` and are picked up on the next agent generation.

```ts
// List catalog + per-room selected flags in one read
const { body: avail } = await app.rest({
  method: 'POST', path: 'rooms.listPrivOSSandboxSkills',
  body: { rid: roomId, useGlobal: true },
});

// Replace: enable exactly these (omit an id to remove it)
await app.rest({
  method: 'POST', path: 'rooms.syncPrivOSSandboxSkills',
  body: { rid: roomId, componentIds: ['skill-a', 'skill-b'] },
});

// Merge: enable this app's agent set without touching anything else
await app.rest({
  method: 'POST', path: 'rooms.syncPrivOSSandboxSkills',
  body: { rid: roomId, addAgentSetIds: ['set-1'] },
});

// Read back the effective selection (covers syncs done in the sandbox Board UI)
const { body: sel } = await app.rest({
  method: 'GET', path: `rooms.getPrivOSSandboxSelection?roomId=${roomId}`,
});
```

## Uploading agent sets (workspace Agent Factory)

A workspace administrator can upload agent-set archives through an app. Because an agent set
carries executable skills, this is the highest-risk scope a marketplace app can request
(risk `critical`, workspace context, interactive user only). Three REST routes
(`app/api/server/v1/agent-privos-sandbox-agent-sets.ts`):

| Path | Method | Purpose |
| --- | --- | --- |
| `agents.sandbox.agentSets.preview` | POST | Submit archives (`{ archives: [{ fileName, base64 }] }`) → the board's projected item list + a `sessionId` |
| `agents.sandbox.agentSets.confirm` | POST | Commit a previewed session (`{ sessionId }`) — all-or-nothing |
| `agents.sandbox.agentSets.delete` | POST | Remove an agent set from the workspace catalog (`{ agentSetId }`). Refused with `{ error: 'agent-set-in-use', referencedRooms: [...] }` while any room still references the set — disable it there first via merge-mode `removeAgentSetIds` |

### Reachability

- **Frontend (`app.rest()`):** all routes reachable with the **`sandbox:agent-sets:upload`**
  scope (`server/services/mcp-rest-allowlist.ts`).
- **Server-side gate:** all routes enforce the native **`manage-privos-agent-sets`**
  (workspace admin) permission, so the scope only widens reachable paths — an app can never
  install or remove a set on its own authority. The board's secure extraction remains the
  sole validation authority; the hub only streams bytes within the board's own limits.
- **App attestation:** the hub attests the calling app itself via an `x-mcp-app-attestation`
  token minted server-side (`server/services/mcp-app-attestation.ts`). The app identity is
  derived from that token — any app id claimed in a request body is ignored.

The [PrivOS demo MCP app](https://github.com/PrivOS-AI/privos-mcp-app-demo/blob/main/src/ui/agent-set-upload-panel.tsx)
contains a reference upload panel, and declares the scope at `context: "workspace"` in its
`privos-app.json`.

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
| `mcpapp.files.getByChannel` | `GET file-management.files.channel/:channelId` |
| `mcpapp.files.get` | `GET file-management.files/:fileId` |
| `mcpapp.files.search` | `GET file-management.files.search/:channelId` |
| `mcpapp.files.update` | `POST file-management.files/:fileId/rename` (or `/update-content`) |
| `mcpapp.files.delete` | `DELETE file-management.files/:fileId` |
| *(file upload — was never a tool)* | `POST file-management.files.upload` |
| *(large file upload)* | `POST file-management.files.upload-chunked-init` → `…upload-chunk` → `…upload-chunked-complete` / `…-cancel` |
| `mcpapp.folders.*` | `file-management.folders.*` (`channel/:channelId`, `:folderId`, create/rename/delete) |
| `mcpapp.lists.*` | `lists.*` (`lists.create`, `lists.list`, `lists.info`, `lists.update`, items via `lists.*`/`items.*`) |
| `mcpapp.lists.addField` / `removeField` | `lists.fields.create` / `lists.fields.delete` (or `lists.addField` / `lists.removeField`) |
| `mcpapp.messages.send` | `POST chat.sendMessage` (or `chat.postMessage`) |
| `mcpapp.messages.getRecent` | `GET channels.messages` / `channels.history` |
| `mcpapp.rooms.get` | `GET channels.info` |
| `mcpapp.rooms.getMembers` | `GET channels.members` |
| `mcpapp.users.get` / `getByIds` | `GET users.info` / `users.list` |
| `mcpapp.bot.sendMessage` / `sendAttachment` | `POST chat.sendMessage` with a bot token (identity = bot user) |

> File **archive extraction** (unzip into the folder tree) has **no** REST endpoint today.
> If needed, add it as a single new REST route (e.g. `file-management.files.extract`) —
> not as a tool — so it stays on the one integration surface.

## Tools with no REST equivalent

These are MCP-runtime / sandbox concerns, not hub resources — keep using them via the
bridge/SDK:

- `mcpapp.context.get` — current room/user context for the iframe session.
- `mcpapp.app.getLocalData` / `setLocalData` / `deleteLocalData` / `clearLocalData` — the
  app's own sandboxed key-value store.
- `mcpapp.db.*` — the app-private collection store (app's own data, not a hub resource).
- `mcpapp.bot.getMe` — runtime bot-token identity check.

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
- **Binary downstreams** (`file-management.files/:id/download`, `/content`) are returned by the
  proxy as a base64 envelope `{ result: { dataBase64, mimeType, fileName, size } }`
  (`server/services/mcp-rest-binary-response.ts`, 32 MiB cap). JSON and `text/*` bodies keep
  the legacy string/JSON `result` unless the app passes `responseType: 'blob'`, which forces
  the envelope and makes the host bridge hand back a `Blob`.
- **Multipart upload** can't tunnel through the JSON proxy, so `host/file.upload` posts
  directly to `file-management.files.upload` as the user; the host gates it on `files:write`
  (the iframe reaches REST only via the trusted host bridge).

### SDK usage (`@privos_ai/app-react`)

```ts
const app = usePrivosApp();

// Read files in the room (needs files:read)
const res = await app.rest({ method: 'GET', path: `file-management.files.channel/${roomId}` });

// Send a message (needs messages:send)
await app.rest({ method: 'POST', path: 'chat.sendMessage', body: { message: { rid: roomId, msg: 'hi' } } });

// Download file bytes as a Blob (needs files:read)
const { body: blob, fileName } = await app.rest({ method: 'GET', path: `file-management.files/${fileId}/download`, responseType: 'blob' });

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
