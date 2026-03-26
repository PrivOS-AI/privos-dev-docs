# App Platform — Overview

MCP-compatible platform for embedding third-party apps in Privos Chat rooms via sandboxed iframes. Apps are MCP servers that expose tools with UI capabilities.

## Key Concepts

- Apps are external MCP servers — Privos connects via **direct HTTP** or **relay WebSocket**
- **Direct apps**: Privos connects via HTTP Streamable to your server
- **Relay apps**: Your app connects via WebSocket relay provided by Privos (ideal for NAT/firewall)
- Tool discovery via `initialize` → `tools/list` JSON-RPC
- Tools with `_meta.ui` render in sandboxed iframes as room tabs
- Apps call Privos resources (lists, files, messages) via `callServerTool()`
- OAuth scope enforcement on every tool call
- Deny-by-default iframe sandbox (no `allow-same-origin`)

## Architecture

### Direct Connection
```
Developer's MCP App Server (external, HTTPS required)
├── /.well-known/mcp/manifest.json
├── /mcp (JSON-RPC 2.0 endpoint)
│
├──────────── Streamable HTTP ──────────
│
Privos Chat Host (MCP Client connects directly)
```

### Relay Connection
```
Admin generates pairing URL → Developer enters during npm start
                              ↓
Developer's MCP App Server → Exchanges token for clientId + clientSecret
                              ↓
                           Obtains OAuth token (POST /oauth/token)
                              ↓
                           WebSocket connection (wss://privos-host/api/v1/mcp-apps.relay)
                              └─ Bearer token in Authorization header
                                 ↓
                                 └──────── JSON-RPC 2.0 over WS ────────────

Privos Chat Host
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
| [Developer Guide](./developer-guide.md) | Direct & relay app setup, build, run |
| [API Reference](./api-reference.md) | REST endpoints, relay WS, MCP tools, scopes |
| [React SDK](./react-sdk-reference.md) | `@privos/app-react` hooks |
| [Admin Guide](./admin-guide.md) | Register direct/relay apps, configure install perms |
| [Security & Data Model](./security-and-data-model.md) | Sandbox, OAuth, relay security, schema |
