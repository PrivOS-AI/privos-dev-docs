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
