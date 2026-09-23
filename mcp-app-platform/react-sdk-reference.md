# App Platform — React SDK Reference

Package: `@privos_ai/app-react@0.4.0`

Thin React wrapper around the MCP `@modelcontextprotocol/ext-apps` SDK with PrivOS-specific convenience hooks. Verify exports against the package's shipped `dist/index.d.ts` if this reference and the installed version ever disagree.

## Hooks

| Hook | Returns | Description |
|------|---------|-------------|
| `usePrivosApp()` | `McpApp` | MCP app instance for `app.rest()`, `app.uploadFile()`, `app.storage`, and `callServerTool()` |
| `usePrivosContext()` | `PrivosContext` | `userId`, `username`, `theme`, `roomId`, `roomName`, `userRoles`, `effectiveScopes?`, `roomSlug?`, `appId?`, `appUrl?`, `themeTokens?` — see note below on extra runtime fields |
| `usePrivosCapability(scope)` | `{ resolved, granted, scope }` | Presentation/degradation helper only — the Hub remains the sole authorization authority, this never gates real access |
| `usePrivosTool(name, args)` | `{ data, loading, error, refetch }` | Generic auto-fetching tool call. Skips the fetch while any arg value is empty/null/undefined |
| `useLists(roomId)` | `{ data, loading, error, refetch }` | Lists in room |
| `useFiles(roomId)` | `{ data, loading, error, refetch }` | Files in room |
| `useRoom(roomId?)` | `{ data, loading, error, refetch }` | Room metadata |
| `useAppChatSurface(options?)` | `{ resolved, supported, isOpen, open, close }` | Claim the Hub's floating AI-chat launcher so this app can render its own chat window instead |
| `useProviderEmbed(url)` | `{ ref, state, reason? }` | Ask the host to hoist a provider iframe (e.g. YouTube, Figma) over a placeholder element — the app's own document cannot embed those origins directly |
| `parseToolResult(result)` | `Record<string, unknown>` | Parse an MCP tool / host-bridge response into a plain object |

There is **no** `useAppDb` or `usePrivosUserToken` hook — see the notes under [No dedicated DB hook](#no-dedicated-db-hook) and [Signed user token](#signed-user-token) below.

## Provider

Wrap your app with `PrivosAppProvider` (backed by `PrivosAppContext`):

```tsx
import { PrivosAppProvider } from '@privos_ai/app-react';

export default function App() {
  return (
    <PrivosAppProvider>
      <MyComponent />
    </PrivosAppProvider>
  );
}
```

`PrivosAppProviderProps`: `children` (required), plus optional `app` (custom `McpApp` implementation — defaults to a PostMessage-based one), `name`, `version`.

## usePrivosApp

Returns the MCP app instance. Use it for `app.rest()` (the preferred way to read/write
hub data — see below) and for direct tool calls to capabilities with no REST equivalent
(`mcpapp.db.*`, `mcpapp.bot.*`):

```tsx
const app = usePrivosApp();

await app.callServerTool({
  name: 'mcpapp.db.create',
  arguments: { collection: 'contacts', data: { name: 'New item' } }
});
```

### app.rest() — call hub REST endpoints

Preferred over resource tools for accessing existing hub data. The host proxies
the call as the current user; the server gates it against the app's granted
scopes (a request to a path the app's scopes don't cover is rejected with 403).

```tsx
const app = usePrivosApp();

// GET — list files in a channel
const files = await app.rest({
  method: 'GET',
  path: 'file-management.files.channel/' + roomId,
  query: { count: 50 },
});

// POST — JSON body
await app.rest({
  method: 'POST',
  path: 'file-management.folders.create',
  body: { channelId: roomId, name: 'Reports' },
});
```

`RestRequestParams`:

| Field | Type | Notes |
|-------|------|-------|
| `method` | `'GET' \| 'POST' \| 'PUT' \| 'PATCH' \| 'DELETE'` | required |
| `path` | `string` | hub REST path after `/api/v1/` |
| `query` | `Record<string, string \| number \| boolean>` | optional querystring |
| `body` | `any` | optional JSON body |
| `timeoutMs` | `number` | optional host-bridge response timeout override (default 10000) |
| `responseType` | `'json' \| 'blob'` | optional; `'blob'` resolves binary downstreams as a `Blob` (see below) |

#### Binary downstreams (file download / content)

The proxy is a JSON route, so a downstream that answers with bytes
(`file-management.files/{fileId}/download`, `/content`) is delivered as a
base64 envelope in `body.result` (`RestBinaryResult`). By default only
non-JSON, non-`text/*` bodies are wrapped: `.md`/`.txt`/`.csv` content keeps
arriving as a plain string in `body.result`, JSON files as parsed JSON.
Requires hub `tenant.240` or later (older hubs decode binary as UTF-8 text).

```json
{ "statusCode": 200, "body": { "result": { "dataBase64": "...", "mimeType": "image/png", "fileName": "shot.png", "size": 2944 } } }
```

Pass `responseType: 'blob'` to force the envelope for every successful
downstream (text and JSON files included) and have the host decode it for you:

```tsx
const { body: blob, fileName } = await app.rest({
  method: 'GET',
  path: `file-management.files/${fileId}/download`,
  responseType: 'blob',
});
const url = URL.createObjectURL(blob);
const a = Object.assign(document.createElement('a'), { href: url, download: fileName ?? 'file' });
a.click();
URL.revokeObjectURL(url);
```

`fileName` comes from the downstream `Content-Disposition`; `blob.type` / `blob.size`
carry `mimeType` / `size`. Without `responseType: 'blob'` decode `dataBase64` yourself.
Base64 is transport-only — persist the `fileId`, not the bytes. Whole file is buffered
in one reply: bodies over 32 MiB are rejected (`statusCode` 400), and large files may
need a higher `timeoutMs`.

### app.uploadFile() — multipart file upload

Uploads a file to file management as the current user. Requires the
**`files:write`** scope in the app manifest (admin-granted).

```tsx
const app = usePrivosApp();

await app.uploadFile({
  channelId: roomId,
  fileName: 'report.pdf',
  base64Data,                  // base64 string or data URI
  mimeType: 'application/pdf',
  // folderId, enableEmbedding, duplicateAction are optional
});
```

`UploadFileParams`:

| Field | Type | Notes |
|-------|------|-------|
| `channelId` | `string` | required — target room |
| `fileName` | `string` | required |
| `base64Data` | `string` | required — base64 content (data URI accepted) |
| `mimeType` | `string` | optional |
| `folderId` | `string` | optional |
| `enableEmbedding` | `boolean` | optional |
| `duplicateAction` | `'replace' \| 'keep_both' \| 'cancel'` | optional |

> The app never exceeds the current user's own permissions — the server still
> enforces per-room ACL and read-only restrictions on every call.

### app.storage — persistent per-app key/value store (SDK ≥ 0.5)

A small string key/value store the app can rely on across sessions. The app
document runs in a **sandboxed opaque origin**, so its own `localStorage` throws
or is wiped between loads. `app.storage` proxies over the host bridge to the
**host's** `localStorage`, where the host writes each value under a per-app
namespace — `mcp-app:{appId}:{key}` — stamping the `appId` itself (never trusting
the iframe payload). One app therefore **cannot read or overwrite** another app's
keys, and a surface with no resolved `appId` is refused rather than sharing a
bucket. No manifest scope is required.

```tsx
const app = usePrivosApp();

await app.storage.set('ui:language', 'vi');
const lang = await app.storage.get('ui:language'); // 'vi' | null
await app.storage.remove('ui:language');
```

| Method | Signature | Notes |
|--------|-----------|-------|
| `get` | `(key: string) => Promise<string \| null>` | `null` when never set |
| `set` | `(key: string, value: string) => Promise<void>` | overwrites; values are strings — serialize objects yourself |
| `remove` | `(key: string) => Promise<void>` | no-op if absent |

**Scope & durability.** Isolated **per app** (by `appId`) and stored **per browser
profile** — it is *not* synced across devices and is *not* per-user on its own
(add a `userId` prefix to the key if two accounts may share a browser profile).
Keep the server (`mcpapp.db`) as the source of truth for anything that must
follow the user across devices, and use `app.storage` as a fast local cache or
for device-local UI preferences.

**Bridge protocol.** `app.storage` sends JSON-RPC requests `host/storage.get`,
`host/storage.set`, `host/storage.remove` (params `{ key, value? }`) to the host
over `postMessage`; the host replies `{ value }` / `{ ok: true }`. An app built
against an SDK older than 0.5 can call these bridge methods directly against a
namespace-aware host.

### app.startMicrophone / app.requestWakeLock — host-brokered devices (SDK ≥ 0.7)

The app document runs in an **opaque origin**, and browsers refuse `getUserMedia`
and the Wake Lock API there, even when the iframe `allow` attribute delegates the
feature (Chrome throws `NotAllowedError` / `SecurityError`). The host does it
instead: it opens the microphone under the hub origin (the browser prompt names
the hub) and streams **mono signed 16-bit PCM** frames back, and it holds the
screen wake lock for you.

```tsx
const app = usePrivosApp();

// Must run inside a click/keypress handler.
const mic = await app.startMicrophone?.({
  sampleRate: 16000,                     // preferred; read mic.sampleRate for the real one
  onData: (chunk: Int16Array) => ws.send(chunk),
  onEnded: (reason) => setRecording(false), // device unplugged / permission revoked
});
if (!mic?.granted) {
  // mic.reason: 'not_declared' | 'user_activation_required' | 'denied' | 'unavailable' | 'unsupported_host'
  // unsupported_host (or no startMicrophone): older hub, fall back to navigator.mediaDevices.getUserMedia
}
// later
mic.stop();

const lock = await app.requestWakeLock?.(); // { granted: true } | { granted: false, reason }
app.releaseWakeLock?.();
```

Rules:

- **Declare it.** The rendered tool must list `microphone` / `screen-wake-lock`
  in `_meta.ui.permissions`, or the host answers `not_declared`.
- **User gesture.** `startMicrophone` must be called from a click/keypress
  handler inside the app (the host checks that the app frame has focus and
  the page has a fresh activation). Otherwise the host answers
  `user_activation_required`.
- **Per-app consent.** The first time an app asks, the hub shows its own
  prompt: "<App> wants to use your microphone — Allow / Block". Allow is
  remembered per user and app; Block answers `denied` for that request only.
  The browser's own permission prompt (for the hub site) comes after it, once.
- **One capture per document.** Starting again replaces the previous capture.
  The host releases the mic and the wake lock when the app document reloads or
  the host unmounts. Switching room tabs keeps both running.
- **Wake lock** is re-acquired by the host whenever the page becomes visible
  again, until `releaseWakeLock()`.
- **Camera is not brokered.** A camera stream cannot be streamed as cheaply as
  PCM, so opaque-origin apps cannot use the camera today.

**Bridge protocol.** Requests: `host/microphone.start` `{ sampleRate?,
echoCancellation?, noiseSuppression?, autoGainControl? }` → `{ granted: true,
streamId, sampleRate, encoding: 'pcm_s16le', channels: 1 }` or `{ granted:
false, reason }`; `host/microphone.stop` `{ streamId }`; `host/wakeLock.request`
→ `{ granted, reason? }`; `host/wakeLock.release`. Host notifications:
`ui/microphone.data` `{ streamId, pcm: ArrayBuffer }` (buffer transferred) and
`ui/microphone.ended` `{ streamId, reason }`. `ui/initialize.hostCapabilities`
carries `microphone: true, wakeLock: true` on hubs that broker.

## usePrivosContext

Subscribes to `HOST_CONTEXT_CHANGED` push + fetches PrivOS-specific context:

```tsx
const { roomId, userId, username, theme, roomName, userRoles, effectiveScopes, roomSlug, appId, appUrl, themeTokens } = usePrivosContext();
```

The TypeScript `PrivosContext` type only declares the fields above, but the hook's
underlying fetch is the same `mcpapp.context.get` tool call documented in
[Tools — Context](./apis/tools-context.md), whose response also carries `userToken`,
`roomType`, `isAgentRoom`, `agentBot`, `defaultBot` when relevant. Those extra fields
are present on the object at runtime but aren't in the type — read them off a cast,
matching the app's own untyped-extras pattern:

```tsx
const ctx = usePrivosContext() as Record<string, any>;
const userToken: string | undefined = ctx.userToken;
const isAgentRoom: boolean | undefined = ctx.isAgentRoom;
```

- `theme` (`'light'` | `'dark'`) — updates in real-time when the user toggles theme
- `themeTokens` (`Record<string, string>`, optional) — the 12 curated `--base-*` workspace design
  tokens for the current mode, re-sent on every mode flip and live admin theme save.
  `PrivosAppProvider` already writes each one onto this document's `<html>` as a CSS custom
  property automatically — see [Theme Inheritance](./theme-inheritance.md) for the full contract
  and how to consume it directly instead of (or alongside) the auto-applied CSS variables.
- `appId` / `roomSlug` / `appUrl` (`basic:information`) — this app's id, the room slug, and the deep-link URL to the app inside the room (`${ROOT_URL}/{channel|direct|group}/{roomSlug}/mcpapp/{appId}`)

Use with a `ThemeProvider` for Auto/Light/Dark mode support. See [Developer Guide — Theme Sync](./developer-guide.md#7-theme-sync-lightdark-mode).

## Signed user token

There is no `usePrivosUserToken()` hook. Read `userToken` off `usePrivosContext()`
(cast as shown above) and forward it to your app **backend**, which verifies it
against the hub JWKS — this proves *who* the caller is without your app being able
to forge it. Never trust a client-supplied `userId` without a token that verifies.

```tsx
const { userToken } = usePrivosContext() as Record<string, any>;
// POST it to your backend, e.g. as Authorization: `Bearer ${userToken}`
```

Backend verification + claims are documented in
[Tools — Context › Signed user identity](./apis/tools-context.md#signed-user-identity).

## usePrivosTool

Generic hook — auto-fetches on mount and when args change. Best for reads from tools with
no REST equivalent (`mcpapp.db.*`):

```tsx
const { data, loading, error, refetch } = usePrivosTool('mcpapp.db.get', { collection, id });
```

**Note:** Skips fetch if any arg value is empty/null/undefined. For hub data (lists, files,
messages, rooms, users), read via `app.rest()` instead.

## Other hooks

- **`usePrivosCapability(scope)`** — returns `{ resolved, granted, scope }` for
  presentation-only degradation (e.g. hide a button before the tool call would 403).
  It never itself authorizes anything.
- **`useAppChatSurface(options?)`** — claims the Hub's floating AI-chat launcher for
  an app that renders its own chat UI. `options.onOpen` / `options.onClose` fire when
  the host asks the app to show/hide its chat window.
- **`useProviderEmbed(url)`** — attach the returned `ref` to a placeholder element;
  the host renders an approved provider (YouTube, Figma, etc.) iframe over it,
  because the app's own sandboxed document cannot embed those origins itself.
- **`parseToolResult(result)`** — normalizes an MCP tool / host-bridge response into
  a plain object; used when calling `app.callServerTool()` directly instead of
  `usePrivosTool()`.

## No dedicated DB hook

There is no `useAppDb()` hook. Call `mcpapp.db.*` tools directly:

```tsx
const app = usePrivosApp();

// Write
await app.callServerTool({ name: 'mcpapp.db.create', arguments: { collection: 'contacts', data: { name: 'Alice' } } });

// Read (auto-fetching)
const { data, loading, error, refetch } = usePrivosTool('mcpapp.db.query', {
  collection: 'contacts',
  where: [{ field: 'name', op: '!=', value: '' }],
  orderBy: [{ field: 'name', direction: 'asc' }],
  limit: 10,
});
```

`mcpapp.db.query` takes a flat `where`/`orderBy` argument object (no chainable query
builder client-side) — see [Database Tools API Reference](./apis/tools-database.md)
for every `mcpapp.db.*` tool's full input schema.

## Pattern: Reads vs Mutations

- **Reads** — use `usePrivosTool` or convenience hooks (`useLists`, `useFiles`, `useRoom`). Auto-fetches.
- **Hub data (REST-first)** — use `app.rest()` to reach existing hub REST endpoints (files, folders, rooms…) as the current user, gated by the app's scopes. Preferred over building dedicated resource tools.
- **File upload** — use `app.uploadFile()` (requires `files:write`).
- **Mutations** — use `usePrivosApp()` to get the app instance, then call `app.callServerTool()` directly in event handlers.
- **Database** — call `mcpapp.db.*` tools via `usePrivosApp().callServerTool()` (writes) and `usePrivosTool()` (reads); there is no dedicated hook.
