# App Platform — Overview

MCP-compatible platform for embedding third-party apps in PrivOS Hub rooms via sandboxed iframes. Apps are MCP servers that expose tools with UI capabilities.

## Key Concepts

- Apps are external MCP servers — PrivOS connects via **direct HTTP** or **relay WebSocket**
- **Direct apps**: PrivOS connects via HTTP Streamable to your server
- **Relay apps**: Your app connects via WebSocket relay provided by PrivOS (ideal for NAT/firewall)
- Tool discovery via `initialize` → `tools/list` JSON-RPC
- Tools with `_meta.ui` render in sandboxed iframes as room tabs
- Apps call PrivOS resources (lists, files, messages, DB) via `callServerTool()`
- OAuth scope enforcement on every tool call (with a small allow-list of scope-free tools like `mcpapp.context.get`)
- Deny-by-default iframe sandbox (no `allow-same-origin`)
- **Optional credential push**: after direct-app connect, the Hub POSTs `{appId, clientId, clientSecret}` to your `/.well-known/mcp/register` endpoint (best-effort, 404 = skipped)

### App kinds (Admin → MCP Apps)

Every installed app shows as one of three kinds to a workspace admin — this is
about how the app *runs*, orthogonal to the direct/relay *transport* above:

- **Standalone Relay** — the developer runs their own app server and connects
  over the relay (registered via Register Relay). Serves its UI live from that
  server; the Hub never stores it.
- **Runtime** — runs on the PrivOS marketplace runtime or a self-hosted
  runtime. The UI is preloaded into the workspace via the
  [signed UI bundle](./ui-bundle.md); tool calls go to the runtime.
- **Instant** — frontend only, no server to run. See [INSTANT apps](./instant-apps.md).

## Architecture

### Direct Connection
```
Developer's MCP App Server (external, HTTPS required)
├── /.well-known/mcp/manifest.json        (GET — required)
├── /mcp                                   (POST — JSON-RPC 2.0, required)
├── /.well-known/mcp/register              (POST — optional, receives credentials)
│
├──────────── Streamable HTTP ──────────
│
PrivOS Hub Host (MCP Client connects directly)
│
└── After connect: best-effort POST credentials → /.well-known/mcp/register
    (lets the app self-configure OAuth credentials for callbacks; failures non-fatal)
```

### Relay Connection
```
Admin generates pairing URL → Developer enters it at `npm run pair`
                              ↓
Developer's MCP App Server → Exchanges token for clientId + clientSecret
                              ↓
                           Obtains OAuth token (POST /oauth/token)
                              ↓
                           WebSocket connection (wss://privos-host/api/v1/mcp-apps.relay)
                              └─ Bearer token in Authorization header
                                 ↓
                                 └──────── JSON-RPC 2.0 over WS ────────────

PrivOS Hub Host
├── Pairing endpoint (generate URL, check status)
├── WebSocket relay (accepts connections from apps)
├── MCP Client (proxies to app via relay)
├── PostMessage Bridge (JSON-RPC 2.0 over postMessage)
├── MinIO storage (.apps/{appId}/ bucket for app icons/assets)
├── File management API (serves app files)
└── App Registry + OAuth scope enforcement
```

## Docs Index

| Doc | Description |
|-----|-------------|
| [Auth & REST Integration](./auth-and-rest-integration.md) | REST-first model, frontend session vs backend bot token, security |
| [Runtime Modes](./runtime-modes.md) | One app across managed / standalone / development — `serveApp`, transport & trust bootstrap |
| [Developer Guide](./developer-guide.md) | Direct & relay app setup, build, DB tutorial, run |
| [The signed UI bundle](./ui-bundle.md) | `bundle-ui`, `ui.distDir`, install/upgrade verification, refusal codes |
| [INSTANT apps](./instant-apps.md) | Frontend-only apps: manifest contract, data in Lists, publish checklist |
| [API Reference](./api-reference.md) | REST endpoints, relay WS, MCP tools, scopes |
| [React SDK](./react-sdk-reference.md) | `@privos_ai/app-react` hooks (`usePrivosApp`, `usePrivosContext`, `useLists`, etc.) |
| [Theme Inheritance](./theme-inheritance.md) | How an app inherits the workspace theme — `--base-*` token broadcast, `hostCapabilities.theme`, SDK auto-apply |
| [Database API](./apis/tools-database.md) | `mcpapp.db.*` tools — schema, CRUD, query, references |
| [Admin Guide](./admin-guide.md) | Register direct/relay apps, configure install perms |
| [Security & Data Model](./security-and-data-model.md) | Sandbox, OAuth, relay security, schema |
