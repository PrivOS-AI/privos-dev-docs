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

**Note:** Privos automatically **exempts** app UI resource endpoints (`/api/v1/mcp-apps.ui-resource`) from `X-Frame-Options` restrictions.

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

## 7. Theme Sync (Light/Dark Mode)

Privos pushes theme changes to apps in real-time via `HOST_CONTEXT_CHANGED` PostMessage.

### How It Works

1. Privos host detects theme change (user toggles light/dark/auto)
2. Host sends `{ method: 'HOST_CONTEXT_CHANGED', params: { theme: 'light' | 'dark' } }` to iframe
3. `usePrivosContext()` hook receives updated `theme` value
4. App sets `data-theme` attribute on `<html>` → CSS variables switch between light/dark palettes

### Recommended Pattern

Use a `ThemeProvider` with three modes:

| Mode | Behavior |
|------|----------|
| **Auto** | Follows Privos host `theme` in real-time |
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

Define light and dark palettes with hardcoded values. Sandboxed iframes cannot access the parent's CSS variables, so use standalone colors that match the Privos palette:

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

| App Variable | Privos Token | Light | Dark |
|-------------|-------------|-------|------|
| `--bg` | `--rcx-color-surface-room` | #F7F8FA | #1F2329 |
| `--bg-card` | `--rcx-color-surface-light` | #FFFFFF | #262931 |
| `--bg-hover` | `--rcx-color-surface-hover` | #F2F3F5 | #2F343D |
| `--text` | `--rcx-color-font-titles-labels` | #1F2329 | #E4E7EA |
| `--text-muted` | `--rcx-color-font-hint` | #6C737A | #9EA2A8 |
| `--border` | `--rcx-color-stroke-light` | #E4E7EA | #353B45 |
| `--accent` | `--rcx-color-button-background-primary-default` | #156FF5 | #095AD2 |
| `--danger` | `--rcx-color-button-background-danger-default` | #EC0D2A | #BB0B21 |

When running inside the Privos iframe, `--rcx-color-*` variables are inherited from the host — colors match automatically. Fallback values used when running standalone.
