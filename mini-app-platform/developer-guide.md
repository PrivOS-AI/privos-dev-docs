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
│   ├── server.ts          # MCP server (Express + JSON-RPC)
│   └── ui/
│       ├── App.tsx         # React app
│       └── main.tsx        # Entry point
├── package.json
└── vite.config.ts
```

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

Use `@privos/app-react` hooks (see [React SDK Reference](./react-sdk-reference.md)):

```tsx
import { PrivosAppProvider, usePrivosContext, useLists } from '@privos/app-react';

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
- **Must NOT** set `X-Frame-Options: DENY` or `SAMEORIGIN` — the app UI renders inside a sandboxed iframe on the Privos domain
- **Must** allow framing via CSP: `frame-ancestors https://your-privos-domain.com` (or `frame-ancestors *` for dev)

**Note:** Privos automatically **exempts** app UI resource endpoints (`/api/v1/mini-apps.ui-resource`) from `X-Frame-Options` restrictions.

### Rendering Modes

1. **MCP Protocol mode** (preferred) — app provides tools with `_meta.ui.resourceUri`, Privos fetches HTML via `resources/read` and renders in sandboxed iframe with PostMessage bridge. No frame headers needed since HTML is loaded via `srcdoc`.

2. **Direct iframe mode** (fallback) — when no MCP tool entry points exist, Privos renders `serverUrl` directly in an iframe. **Requires server to allow framing.**

### Endpoints Required

| Endpoint | Required | Purpose |
|----------|----------|---------|
| `GET /.well-known/mcp/manifest.json` | Yes | App metadata |
| `POST /mcp` | For MCP mode | JSON-RPC 2.0 |
| `GET /` (or custom path) | For direct iframe | Embeddable HTML UI |

### Common Issues
- **Blank iframe**: Check `X-Frame-Options` and CSP `frame-ancestors` headers
- **Mixed content**: If Privos runs on HTTPS, app server must also use HTTPS
- **CORS**: Not needed for iframe embedding, but needed if app calls Privos API directly
