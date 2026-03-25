# App Platform — Overview

MCP-compatible platform for embedding third-party apps in Privos Chat rooms via sandboxed iframes. Apps are MCP servers that expose tools with UI capabilities.

## Key Concepts

- Apps are external MCP servers — Privos connects as MCP client
- Tool discovery via `initialize` → `tools/list` JSON-RPC (Streamable HTTP)
- Tools with `_meta.ui` render in sandboxed iframes as room tabs
- Apps call Privos resources (lists, files, messages) via `callServerTool()`
- OAuth scope enforcement on every tool call
- Deny-by-default iframe sandbox (no `allow-same-origin`)

## Architecture

```
Developer's MCP App Server (external)
├── /.well-known/mcp/manifest.json (metadata: name, version, author)
├── /mcp (JSON-RPC 2.0 endpoint)
│   ├── initialize → server capabilities
│   ├── tools/list → tools with _meta.ui
│   └── resources/read → ui:// HTML content
│
│       ↕ Streamable HTTP (JSON-RPC 2.0)
│
Privos Chat Host
├── MCP Client (connects to app servers, discovers tools)
├── App Registry (mcp_apps + mcp_app_installations collections)
├── PostMessage Bridge (JSON-RPC 2.0 over postMessage)
│   ├── Sandboxed iframe rendering (deny-by-default)
│   ├── Tool call proxying (app→host→registry→host→app)
│   └── HOST_CONTEXT_CHANGED push (roomId, userId, theme)
├── MCP Tool Registry (16+ Privos tools apps can call)
└── OAuth scope enforcement per tool call
```

## Docs Index

| Doc | Description |
|-----|-------------|
| [Developer Guide](./developer-guide.md) | Scaffold, build, and run an app |
| [API Reference](./api-reference.md) | REST endpoints, MCP tools, scopes |
| [React SDK](./react-sdk-reference.md) | `@privos/app-react` hooks |
| [Admin Guide](./admin-guide.md) | Register, configure, install apps |
| [Security & Data Model](./security-and-data-model.md) | Sandbox, scopes, collections, file structure |
