# App Platform — Developer Guide

## 1. Scaffold

```bash
npx create-privos-mcp-app my-app
cd my-app && npm install
```

Generated structure:
```
my-app/
├── src/
│   ├── server.ts          # Express app wired through @privos_ai/app-server's
│   │                       # createDirectRouter (handles auth, dispatch, UI
│   │                       # resource serving — you only implement tools/list
│   │                       # and your own tool-call routing)
│   └── ui/
│       ├── App.tsx         # React app (@privos_ai/app-react)
│       └── main.tsx        # Entry point
├── privos-app.json         # Manifest (schemaVersion 2+; see §5)
├── package.json
└── vite.config.ts
```

The server no longer hand-rolls JSON-RPC dispatch — `@privos_ai/app-server`
(`createDirectRouter`, `verifyPrivosUser`, and, in production, the workload
identity client) owns the protocol and auth plumbing. For a fuller
production-grade reference — workload identity, relay pairing, license
gating, runtime dispatch v3 — read the
[`privos-mcp-app-demo`](https://github.com/PrivOS-AI/privos-mcp-app-demo)
repository rather than hand-building the server from scratch.

## 2. Define Tools

In `src/server.ts`, the `tools/list` handler defines app capabilities:

```typescript
tools: [{
  name: 'my_dashboard',
  title: 'Dashboard',
  description: 'Interactive dashboard',
  inputSchema: { type: 'object', properties: { roomId: { type: 'string' } } },
  _meta: {
    ui: {
      resourceUri: 'ui://my-app/dashboard.html',
      permissions: [],                              // camera, microphone, etc.
      csp: { 'script-src': ['https://cdn.example.com'] }
    }
  }
}]
```

- Tools **with** `_meta.ui` → iframe rendering in room tab
- Tools **without** `_meta.ui` → text-only (no UI)

## 3. Build the UI

Use `@privos_ai/app-react` hooks (see [React SDK Reference](./react-sdk-reference.md)):

```tsx
import { PrivosAppProvider, usePrivosContext, useLists } from '@privos_ai/app-react';

function Dashboard() {
  const ctx = usePrivosContext();
  const { data: lists, loading } = useLists(ctx.roomId);
  return <div>{lists?.map(l => <div key={l._id}>{l.name}</div>)}</div>;
}

export default function App() {
  return <PrivosAppProvider><Dashboard /></PrivosAppProvider>;
}
```

## 4. Run Dev Server

```bash
npm run dev
# MCP server → http://localhost:3001
# UI dev    → http://localhost:5173
```

## 5. Manifest Format

`/.well-known/mcp/manifest.json`:

```json
{
  "name": "com.example.task-tracker",
  "version": "1.0.0",
  "title": "Task Tracker",
  "description": "Track tasks in room tabs",
  "icon": "/icon.png",
  "author": {
    "name": "John Doe",
    "email": "john@example.com",
    "website": "https://example.com"
  },
  "homepage": "https://example.com",
  "repository": "https://github.com/example/task-tracker"
}
```

**Required:** `name`, `version`

**Icon:** 96x96, PNG or SVG. Relative URLs resolved against server base URL.

**Author:** supports string (`"John Doe"`) or object (`{ name, email?, website? }`). String auto-converts to `{ name }`.

## 6. App Server Requirements

### HTTP Headers
- **Must NOT** set `X-Frame-Options: DENY` or `SAMEORIGIN` — the app UI renders inside a sandboxed iframe on the PrivOS domain
- **Must** allow framing via CSP: `frame-ancestors https://your-privos-domain.com` (or `frame-ancestors *` for dev)

**Note:** PrivOS automatically **exempts** app UI resource endpoints (`/api/v1/mcp-apps.ui-resource`) from `X-Frame-Options` restrictions.

### Rendering Modes

1. **MCP Protocol mode** (preferred) — app provides tools with `_meta.ui.resourceUri`, PrivOS fetches HTML via `resources/read` and renders in sandboxed iframe with PostMessage bridge. No frame headers needed since HTML is loaded via `srcdoc`.

2. **Direct iframe mode** (fallback) — when no MCP tool entry points exist, PrivOS renders `serverUrl` directly in an iframe. **Requires server to allow framing.**

### Endpoints Required

| Endpoint | Required | Purpose |
|----------|----------|---------|
| `GET /.well-known/mcp/manifest.json` | Yes | App metadata |
| `POST /mcp` | For MCP mode | JSON-RPC 2.0 |
| `GET /` (or custom path) | For direct iframe | Embeddable HTML UI |
| `POST /.well-known/mcp/register` | Optional, OAuth-credential apps only | Receive `{appId, clientId, clientSecret}` after connect for auto-config |

**Note on credentials:** `/.well-known/mcp/register` applies to apps that
authenticate back to the Hub with an OAuth client secret. Managed Marketplace
apps built on `@privos_ai/app-server` instead use **workload identity** — a
per-installation Unix socket the SDK exchanges for short-lived, sender-constrained
tokens, with no client secret and nothing to auto-receive here. See the
[`privos-mcp-app-demo`](https://github.com/PrivOS-AI/privos-mcp-app-demo) README's
"Runtime trust model" section for that path. The two contract endpoints above
(manifest + `POST /mcp`) apply either way.

### Optional: Auto-receive credentials via `/.well-known/mcp/register`

Right after `mcp-apps.connect` succeeds, the Hub does a **best-effort POST** with the credentials your app needs to call back into PrivOS. Implementing this endpoint means **zero manual copy/paste** for the admin. This applies only to the OAuth-credential path described above — not to workload-identity apps.

**Request body** (sent by Hub):

```json
{
  "appId": "my-app",
  "clientId": "client_abc",
  "clientSecret": "secret_xyz",
  "timestamp": "2026-05-25T10:30:00.000Z"
}
```

**Reference implementation (Express)**:

```typescript
import express from 'express';
import fs from 'node:fs/promises';

const EXPECTED_HUB_HOST = process.env.EXPECTED_HUB_HOST; // e.g. "chat.privos.com"

app.post('/.well-known/mcp/register', express.json(), async (req, res) => {
  const { appId, clientId, clientSecret, timestamp } = req.body || {};

  if (!appId || !clientId || !clientSecret) {
    return res.status(400).json({ error: 'Missing required fields' });
  }

  // Optional: pin expected hub host to mitigate accidental cross-hub registration
  if (EXPECTED_HUB_HOST && req.hostname !== EXPECTED_HUB_HOST) {
    // For inbound, this checks the Host header — combine with TLS pinning at proxy level
    return res.status(403).json({ error: 'Unexpected hub' });
  }

  // Persist credentials (env file, secret manager, KV store, etc.)
  await fs.writeFile('.env.runtime', [
    `MCP_APP_ID=${appId}`,
    `MCP_CLIENT_ID=${clientId}`,
    `MCP_CLIENT_SECRET=${clientSecret}`,
    `MCP_REGISTERED_AT=${timestamp}`,
  ].join('\n'));

  // Idempotent: same appId always overwrites — Hub may retry on rotation
  return res.status(200).json({ ok: true });
});
```

**Hub behavior on your response**:

| Your response | What Hub does |
|---|---|
| 2xx | Success — credentials delivered |
| 404 | Skipped — assumes you intentionally do not support auto-config (no warning logged) |
| 4xx (other) / 5xx / timeout / network error | Logged as warning, connect still succeeds — admin can copy credentials from the connect response |

**Trust model**: this push has no signature — security relies on HTTPS and the developer trusting the configured `serverUrl`. For details, see [security-and-data-model.md](./security-and-data-model.md#credential-push-to-app-server-direct-apps).

### Common Issues
- **Blank iframe**: Check `X-Frame-Options` and CSP `frame-ancestors` headers
- **Mixed content**: If PrivOS runs on HTTPS, app server must also use HTTPS
- **CORS**: Not needed for iframe embedding, but needed if app calls PrivOS API directly

## 7. Theme Sync (Light/Dark Mode)

PrivOS pushes theme changes to apps in real-time via `HOST_CONTEXT_CHANGED` PostMessage.

### How It Works

1. PrivOS host detects theme change (user toggles light/dark/auto)
2. Host sends `{ method: 'HOST_CONTEXT_CHANGED', params: { theme: 'light' | 'dark' } }` to iframe
3. `usePrivosContext()` hook receives updated `theme` value
4. App sets `data-theme` attribute on `<html>` → CSS variables switch between light/dark palettes

### Recommended Pattern

Use a `ThemeProvider` with three modes:

| Mode | Behavior |
|------|----------|
| **Auto** | Follows PrivOS host `theme` in real-time |
| **Light** | Forces light regardless of host |
| **Dark** | Forces dark regardless of host |

```tsx
const { theme } = usePrivosContext();
<ThemeProvider hostTheme={theme}>
  <App />
</ThemeProvider>
```

The `ThemeProvider` sets `data-theme="light"` or `data-theme="dark"` on `<html>`. CSS variables handle everything else — no inline style overrides needed.

### CSS Variables

Define light and dark palettes with hardcoded values. Sandboxed iframes cannot access the parent's CSS variables, so use standalone colors that match the PrivOS palette:

```css
:root, [data-theme="light"] {
  --bg: #F7F8FA;
  --text: #1F2329;
  --accent: #156FF5;
  --border: #E4E7EA;
}

[data-theme="dark"] {
  --bg: #080d0f;
  --text: #E4E7EA;
  --accent: #095AD2;
  --border: #353B45;
}

body { background: var(--bg); color: var(--text); }
```

### Key Token Mappings

| App Variable | PrivOS Token | Light | Dark |
|-------------|-------------|-------|------|
| `--bg` | `--rcx-color-surface-room` | #F7F8FA | #1F2329 |
| `--bg-card` | `--rcx-color-surface-light` | #FFFFFF | #262931 |
| `--bg-hover` | `--rcx-color-surface-hover` | #F2F3F5 | #2F343D |
| `--text` | `--rcx-color-font-titles-labels` | #1F2329 | #E4E7EA |
| `--text-muted` | `--rcx-color-font-hint` | #6C737A | #9EA2A8 |
| `--border` | `--rcx-color-stroke-light` | #E4E7EA | #353B45 |
| `--accent` | `--rcx-color-button-background-primary-default` | #156FF5 | #095AD2 |
| `--danger` | `--rcx-color-button-background-danger-default` | #EC0D2A | #BB0B21 |

When running inside the PrivOS iframe, `--rcx-color-*` variables are inherited from the host — colors match automatically. Fallback values used when running standalone.

---

## 8. Host PostMessage Bridge

Apps running in a sandboxed iframe communicate with the PrivOS host via `window.postMessage` using JSON-RPC 2.0. `@privos_ai/app-react` wraps most of this, but these are the raw host-exposed methods:

| Method | Direction | Params | Returns | Purpose |
|--------|-----------|--------|---------|---------|
| `ui/initialize` | Host → app | `{ hostCapabilities }` | — | First message after iframe load — signals host is ready |
| `HOST_CONTEXT_CHANGED` | Host → app | `{ userId, roomId, roomName, theme, surfaceColor, ... }` | — | Push host context (sent once after `ui/initialize`, then on any change) |
| `tools/call` | App → host | `{ name, arguments }` | tool-specific | Invoke a PrivOS MCP tool (e.g. `mcpapp.bot.sendMessage`, `mcpapp.db.create`). For lists/files/messages/rooms/users, use the REST API via `app.rest()` instead. |
| `rooms/upload` | App → host | `{ roomId, fileName, base64Data, mimeType?, description?, uploadOnly? }` | `{ message: ... }` (default) **or** `{ file: { _id, name, type, size, url } }` (when `uploadOnly: true`) | Upload a file into a room using the host user's credentials. Default behavior also posts a user-authored message. Pass `uploadOnly: true` to upload only (no message) — useful when you want a bot to be the sole author of the resulting message via `mcpapp.bot.sendAttachment`. |
| `OPEN_LINK` | App → host | `{ url }` | — | Open external URL in new tab (`noopener,noreferrer`) |
| `host/storage.get` | App → host | `{ key }` | `{ value }` | Read persistent value |
| `host/storage.set` | App → host | `{ key, value }` | `{ ok: true }` | Write persistent value |
| `host/chatContext.set` | App → host | `{ context }` | `{ ok: true }` | Override AI Chat context string for this app's room tab |
| `SIZE_CHANGED` | App → host | `{ width?, height? }` | — | Hint to host about content size (currently a noop — host lays out iframe externally) |
| `REQUEST_TEARDOWN` | App → host | — | — | Tell host the app is unmounting; host flips `isReady` back to `false` |

### Persistent Storage

`host/storage.get` / `host/storage.set` proxy to the host's `localStorage` under a sandboxed `mcp-app:` prefix. Apps cannot read or write host keys (e.g. Meteor login tokens).

- Keys are stored as `mcp-app:<your-key>` in host storage — apps see only their key, not the prefix.
- Values are strings; serialize JSON yourself (`JSON.stringify` before `set`, `JSON.parse` after `get`).
- Storage is scoped to the PrivOS host origin and shared across app instances on that origin. There is **no per-app isolation yet** — collisions between apps on the same key are possible. Use a unique prefix such as `<appId>:`.
- Missing keys return `{ value: null }`.

Example (raw):

```js
// Write
parent.postMessage({ jsonrpc: '2.0', id: 1, method: 'host/storage.set', params: { key: 'my-app:prefs', value: JSON.stringify({ theme: 'dark' }) } }, '*');
// Read
parent.postMessage({ jsonrpc: '2.0', id: 2, method: 'host/storage.get', params: { key: 'my-app:prefs' } }, '*');
```

### Uploading files to a room (`rooms/upload`)

`rooms/upload` proxies to `/api/v1/rooms.upload/{roomId}` using the **current user's** credentials. Two modes:

- **Default** — uploads file **and** posts a user-authored message. Use when the user is the intended author (e.g. a "Save report" button).
- **`uploadOnly: true`** — uploads file only and returns the upload record; no message is posted. Use when you want a **bot** to be the visible author via `mcpapp.bot.sendAttachment` with `source.fileId`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `roomId` | string | yes | Target room — user must be a member |
| `fileName` | string | yes | Filename including extension |
| `base64Data` | string | yes | Base64-encoded file bytes — **without** the `data:...,` prefix |
| `mimeType` | string | no | Defaults to `application/octet-stream` |
| `description` | string | no | Caption for the uploaded message (ignored when `uploadOnly: true`) |
| `uploadOnly` | boolean | no | Skip posting a message; return upload record only |

Response when `uploadOnly` is unset (default):

```json
{
  "message": {
    "_id": "msg_abc",
    "rid": "room_xyz",
    "file": { "_id": "upload_111", "name": "chart.png", "type": "image/png" },
    "files": [{ "_id": "upload_111" }],
    "attachments": [...]
  }
}
```

Response when `uploadOnly: true`:

```json
{
  "file": {
    "_id": "upload_111",
    "name": "chart.png",
    "type": "image/png",
    "size": 12345,
    "url": "/file-upload/upload_111/chart.png"
  }
}
```

`url` is a relative path served by the host. Authenticated users can fetch it directly; pass it as `source.fileUrl` in `mcpapp.bot.sendAttachment` if you prefer that path over `source.fileId`.

#### Pattern: bot posts a large file generated by the app

For files that exceed the 8 MB cap on `mcpapp.bot.sendAttachment`'s `source.base64Data`:

```typescript
// 1. Upload via bridge with uploadOnly to avoid a user-authored message
const upload = await app.callHostMethod('rooms/upload', {
  roomId,
  fileName: 'export.zip',
  base64Data,            // can be multi-MB; bridge converts to multipart
  mimeType: 'application/zip',
  uploadOnly: true,
});

// 2. Bot posts the message referencing the upload
await app.callServerTool({
  name: 'mcpapp.bot.sendAttachment',
  arguments: {
    botToken,
    roomId,
    type: 'document',
    source: { fileId: upload.file._id },
    text: 'Your export is ready',
  },
});
```

Result: exactly **one message** in the room, authored by the bot.

Raw postMessage version:

```js
parent.postMessage({
  jsonrpc: '2.0',
  id: 99,
  method: 'rooms/upload',
  params: { roomId: '...', fileName: 'data.csv', base64Data: btoa('a,b\n1,2'), mimeType: 'text/csv' },
}, '*');

window.addEventListener('message', (e) => {
  if (e.data?.id === 99) console.log('uploaded', e.data.result);
});
```

### AI Chat Context Override

When an MCP app is open in a room tab, PrivOS renders a floating AI Chat button alongside the app. The chat carries a context string that the AI agent reads to understand what the user is working on.

- **Default context** (when the app sets nothing): `MCP App: <appName> [<mcpAppId>]`.
- **App override**: send `host/chatContext.set` with `{ context: "<your string>" }` to replace the default. Send `{ context: "" }` to revert.

**Persistence rules:**
- The override is scoped per MCP App tab and persists across tab switches inside the room (e.g. switch to messages, back to the app — your value is still there).
- It resets to default when the room is left or the page is refreshed (the tab is "re-opened").
- Each app tab has its own context — overriding context in App A does not affect App B.

Example:

```js
// Override the AI chat context to reflect what the user is currently editing.
parent.postMessage({
  jsonrpc: '2.0',
  id: 1,
  method: 'host/chatContext.set',
  params: { context: 'Editing invoice INV-2026-0042 (status: draft)' },
}, '*');

// Revert to the default `MCP App: <name> [<id>]`.
parent.postMessage({
  jsonrpc: '2.0',
  id: 2,
  method: 'host/chatContext.set',
  params: { context: '' },
}, '*');
```

When the user opens the AI chat from the MCP app tab, the context appears as an insertable chip (cube icon) above the input — clicking **Insert** attaches it to the next message sent to the agent.

---

## 9. Relay Apps (WebSocket Connection)

For apps behind NAT, firewall, or private networks, use the relay connection type. Your app connects outbound to PrivOS via WebSocket with OAuth credentials obtained through a one-click pairing flow.

### Setup — Auto-Pairing Flow

1. **Admin generates pairing URL** (see [Admin Guide](./admin-guide.md))
   - URL: `https://chat.privos.com/pair?token=pair_abc_123xyz` (1-hour expiry)
   - Share URL with app developer

2. **Developer pairs the app**:
   ```bash
   npm run pair      # or: pnpm pair
   # Enter the one-time pairing URL from Hub Admin:
   # Paste: https://chat.privos.com/pair?token=pair_abc_123xyz
   ```

   The command takes no arguments — it asks for the URL, so the URL never lands in shell
   history. On success it starts the app for you, continuing into `start` through whichever
   package manager you invoked it with. Pairing is a one-time step: every later restart uses
   what pairing persisted and needs no URL.

   **A v3 standalone app pairs TWICE.** It announces its `privos-app.json` over the pairing
   socket, so no admin ever handles the manifest file. The first run only REGISTERS the app —
   the Hub grants nothing, reports `awaitingApproval`, and the app neither receives dispatch
   trust nor starts, because trust belongs to the generation an approved permission ceiling
   creates. An admin approves the declared permissions in Hub Admin > Apps, then the developer
   runs `npm run pair` again with a fresh URL from that app's own settings; that second run
   receives trust and starts the app.

3. **App exchanges token for credentials**:
   - Sends `pair_token` to `POST /api/v1/mcp-apps.pair-status`
   - Receives `clientId`, `clientSecret`, `relayUrl`
   - Auto-saves to `.env` (or creates `.env.local`)

4. **App obtains OAuth token and connects**:
   ```bash
   POST /oauth/token
   grant_type=client_credentials
   client_id=CLIENT_ID
   client_secret=CLIENT_SECRET

   # Returns: { "access_token": "...", "expires_in": 3600 }
   ```

5. **App connects to relay WebSocket**:
   ```typescript
   const token = await getOAuthToken();
   const ws = new WebSocket('wss://chat.privos.com/api/v1/mcp-apps.relay', {
     headers: { Authorization: `Bearer ${token}` }
   });

   ws.on('message', (data) => {
     const msg = JSON.parse(data);
     handleMcpMessage(msg);
   });
   ```

### Re-pairing after an uninstall/reinstall

Admin uninstall never deletes the app's catalog entry — it's retained
(`suspended`) so the admin can re-approve later. But re-approval provisions a
**brand-new generation and a brand-new OAuth client id/secret**. The old
credentials are dead: dispatch refuses commands against an uninstalled
generation.

Your app's persisted identity file (default `./privos-standalone-identity.json`,
override via `PRIVOS_STANDALONE_IDENTITY_FILE`) is written with `wx` — it
refuses to overwrite a file that's already there. Before running `npm run pair`
again for the new pairing URL, **delete the old identity file**, or pairing
fails with `IDENTITY_FILE_ALREADY_EXISTS`. There is no automatic migration
between generations; re-pairing from scratch is the only path back after an
uninstall.

### Manual Credential Entry (Fallback)

If the pairing URL is lost, retrieve credentials from admin:
- `CLIENT_ID` and `CLIENT_SECRET` stored in admin portal
- Create `.env` manually:
  ```
  RELAY_URL=wss://chat.privos.com/api/v1/mcp-apps.relay
  CLIENT_ID=client_abc
  CLIENT_SECRET=secret_xyz
  ```

### Handling MCP Messages

### Handling MCP Messages

Relay receives the same JSON-RPC 2.0 messages as direct apps, and can also receive server-initiated `tools/call` requests when the relay app invokes PrivOS server tools via `callServerTool()`:

```typescript
function handleMcpMessage(msg) {
  if (msg.method === 'initialize') {
    // Respond with server capabilities
    ws.send(JSON.stringify({
      jsonrpc: '2.0',
      id: msg.id,
      result: {
        protocolVersion: '2025-01-15',
        capabilities: { tools: {} },
        serverInfo: { name: 'my-app', version: '1.0' }
      }
    }));
  } else if (msg.method === 'tools/list') {
    // Respond with tool list
    ws.send(JSON.stringify({
      jsonrpc: '2.0',
      id: msg.id,
      result: { tools: [/* ... */] }
    }));
  } else if (msg.method === 'resources/read') {
    // Respond with UI HTML for resource URI
    ws.send(JSON.stringify({
      jsonrpc: '2.0',
      id: msg.id,
      result: { contents: [{ uri: msg.params.uri, mimeType: 'text/html', text: '<html>...</html>' }] }
    }));
  } else if (msg.method === 'tools/call') {
    // Handle tool invocation
    handleToolCall(msg).then(result => {
      ws.send(JSON.stringify({
        jsonrpc: '2.0',
        id: msg.id,
        result
      }));
    });
  }
}
```

### Auto-Reconnect Pattern

```typescript
const BACKOFF_MS = [1000, 2000, 5000, 10000, 30000]; // exponential backoff
let reconnectAttempt = 0;

async function connectRelay() {
  try {
    const token = await getOAuthToken();
    const ws = new WebSocket(relayUrl, { headers: { Authorization: `Bearer ${token}` } });

    ws.on('open', () => {
      reconnectAttempt = 0; // reset backoff
      console.log('Relay connected');
    });

    ws.on('error', (err) => {
      console.error('Relay error:', err);
    });

    ws.on('close', () => {
      scheduleReconnect();
    });

    return ws;
  } catch (err) {
    console.error('Failed to get OAuth token:', err);
    scheduleReconnect();
  }
}

function scheduleReconnect() {
  const delay = BACKOFF_MS[Math.min(reconnectAttempt++, BACKOFF_MS.length - 1)];
  console.log(`Reconnecting in ${delay}ms...`);
  setTimeout(connectRelay, delay);
}

// Start initial connection
connectRelay();
```

### Relay App Architecture

**No HTTP server needed.** Relay apps:
- Use `npm run pair` once to pair, then `npm start` to connect to PrivOS via WebSocket
- Serve manifest via Node.js/Express embedded in the relay app package
- UI is built with Vite, compiled to static HTML, sent via `resources/read` JSON-RPC
- Admin portal inlines UI via data URI in pairing metadata
- App icon sent via pairing response AND in initialize `serverInfo.icon` for refresh

### Relay App Manifest

Include `/.well-known/mcp/manifest.json` served locally:

```json
{
  "name": "com.example.relay-app",
  "version": "1.0.0",
  "title": "My Relay App",
  "description": "App that runs behind NAT via relay",
  "icon": "/icon.png",
  "author": { "name": "Your Name" }
}
```

---

## 10. App Database (mcpapp.db.*)

Give your app a full database with schema registration, CRUD, queries, references, and migrations — all without managing your own database server.

> **When to use:** App needs to store structured, queryable data (CRM contacts, inventory, form submissions, etc.). For simple tabular data, consider `mcpapp.lists.*` instead.

**Request scopes** in your manifest: `db:read`, `db:write`, `db:schema:read`, `db:schema:write` as needed.

There is no `useAppDb()` hook — call `mcpapp.db.*` tools via `usePrivosApp().callServerTool()` (writes) and `usePrivosTool()` (reads). Schema registration, CRUD, the query args shape (flat `where`/`orderBy`, not a chainable builder), references/population, cascade-delete rules, data scope (`global` vs `room`), and limits are all documented once in the
[Database Tools API Reference](./apis/tools-database.md) — including a full call-by-call tutorial — rather than duplicated here.
