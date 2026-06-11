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
