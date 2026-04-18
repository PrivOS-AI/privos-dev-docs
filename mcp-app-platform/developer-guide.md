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

---

## 8. Relay Apps (WebSocket Connection)

For apps behind NAT, firewall, or private networks, use the relay connection type. Your app connects outbound to Privos via WebSocket with OAuth credentials obtained through a one-click pairing flow.

### Setup — Auto-Pairing Flow

1. **Admin generates pairing URL** (see [Admin Guide](./admin-guide.md))
   - URL: `https://chat.privos.com/pair?token=pair_abc_123xyz` (1-hour expiry)
   - Share URL with app developer

2. **Developer starts app with pairing URL**:
   ```bash
   npm start
   # ℹ️  Enter pairing URL (or press Enter to skip):
   # Paste: https://chat.privos.com/pair?token=pair_abc_123xyz
   ```

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

Relay receives the same JSON-RPC 2.0 messages as direct apps, and can also receive server-initiated `tools/call` requests when the relay app invokes Privos server tools via `callServerTool()`:

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
- Use `npm start` to connect to Privos via WebSocket
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

## 9. App Database (privos.db.*)

Give your app a full database with schema registration, CRUD, queries, references, and migrations — all without managing your own database server.

> **When to use:** App needs to store structured, queryable data (CRM contacts, inventory, form submissions, etc.). For simple tabular data, consider `privos.lists.*` instead.

### Quick Start

**Step 1 — Request scopes** in your manifest:

```json
{
  "name": "com.example.crm",
  "scopes": ["db:read", "db:write", "db:schema:read", "db:schema:write"]
}
```

**Step 2 — Register schema on first load:**

```tsx
import { useAppDb } from '@privos/app-react';
import { useEffect, useRef } from 'react';

function App() {
  const db = useAppDb();
  const initialized = useRef(false);

  useEffect(() => {
    if (initialized.current) return;
    initialized.current = true;

    // Register collections (idempotent — errors if already exists, so catch)
    db.registerCollection('companies', [
      { name: 'name', type: 'string', required: true, maxLength: 200 },
      { name: 'industry', type: 'string', enum: ['tech', 'finance', 'healthcare', 'other'] },
      { name: 'size', type: 'number', min: 1 },
    ]).catch(() => {}); // already registered — ok

    db.registerCollection('contacts', [
      { name: 'name', type: 'string', required: true },
      { name: 'email', type: 'string', maxLength: 254 },
      { name: 'phone', type: 'string' },
      { name: 'tags', type: 'array', itemType: 'string', maxItems: 10 },
      { name: 'company', type: 'reference', refCollection: 'companies', onDelete: 'set-null' },
    ], [
      { fields: { email: 1 }, unique: true },
    ]).catch(() => {});
  }, []);

  return <ContactList />;
}
```

**Step 3 — CRUD operations:**

```tsx
function ContactList() {
  const db = useAppDb();
  const [contacts, setContacts] = useState([]);

  // Load contacts
  const loadContacts = async () => {
    const { records } = await db.query('contacts')
      .where('tags', 'array-contains', 'vip')
      .orderBy('name', 'asc')
      .limit(50)
      .execute();
    setContacts(records);
  };

  // Create
  const addContact = async () => {
    const contact = await db.create('contacts', {
      name: 'Alice Nguyen',
      email: 'alice@example.com',
      tags: ['vip', 'partner'],
      company: { _ref: 'companies/507f1f77bcf86cd799439011' },
    });
    setContacts(prev => [...prev, contact]);
  };

  // Update
  const updateContact = async (id, data) => {
    const updated = await db.update('contacts', id, data);
    setContacts(prev => prev.map(c => c._id === id ? updated : c));
  };

  // Delete (soft-delete)
  const deleteContact = async (id) => {
    await db.delete('contacts', id);
    setContacts(prev => prev.filter(c => c._id !== id));
  };

  useEffect(() => { loadContacts(); }, []);

  return (
    <ul>
      {contacts.map(c => (
        <li key={c._id}>
          {c.name} — {c.email}
          <button onClick={() => deleteContact(c._id)}>Delete</button>
        </li>
      ))}
      <button onClick={addContact}>Add Contact</button>
    </ul>
  );
}
```

### Data Scope: Global vs Room

| Scope | Collection Name | Use Case |
|-------|----------------|----------|
| `global` (default) | `app_{appId}_{collection}` | Shared data across all rooms (user profiles, settings, catalogs) |
| `room` | `app_{appId}_{roomId}_{collection}` | Per-room isolation (project tasks, team notes, channel polls) |

```tsx
// Global — same data in every room
db.registerCollection('products', [
  { name: 'sku', type: 'string', required: true },
  { name: 'price', type: 'number' },
]);

// Room-scoped — each room has its own data
db.registerCollection('poll_votes', [
  { name: 'userId', type: 'string', required: true },
  { name: 'choice', type: 'string', required: true },
], undefined, 'room');
```

### Query Builder

Chainable, Firestore-style query builder:

```tsx
// Basic filter + sort + pagination
const page1 = await db.query('contacts')
  .where('industry', '==', 'tech')
  .where('size', '>=', 50)
  .orderBy('name', 'asc')
  .limit(20)
  .offset(0)
  .execute();
// → { records: [...], total: 142 }

// Count without fetching records
const { count } = await db.query('contacts')
  .where('tags', 'array-contains', 'vip')
  .count();

// Aggregation
const stats = await db.aggregate('contacts', 'count');
const avgSize = await db.aggregate('companies', 'avg', 'size',
  [{ field: 'industry', op: '==', value: 'tech' }]
);
const byIndustry = await db.aggregate('companies', 'count', undefined,
  undefined, 'industry'  // groupBy
);
```

**Available operators:** `==`, `!=`, `>`, `<`, `>=`, `<=`, `in`, `not-in`, `array-contains`, `array-contains-any`

### References & Population

Link documents across collections:

```tsx
// 1. Create a company
const company = await db.create('companies', { name: 'ACME Inc', industry: 'tech' });

// 2. Create a contact referencing that company
const contact = await db.create('contacts', {
  name: 'Bob',
  company: { _ref: `companies/${company._id}` },
});

// 3. Populate — resolve references to full documents (1-level deep)
const populated = await db.populate('contacts', [contact._id], ['company']);
// populated[0].company → { _id: '...', name: 'ACME Inc', industry: 'tech', ... }
```

**Cascade delete rules** — set on the reference field at schema registration:

| Rule | Behavior when referenced doc is deleted |
|------|----------------------------------------|
| `cascade` | Auto soft-delete all referencing records |
| `set-null` | Set reference field to null |
| `restrict` | Block deletion if references exist |
| `no-action` | Do nothing (orphan refs allowed) |

### Limits & Security

| Constraint | Value |
|-----------|-------|
| Collections per app | 20 max |
| Query result size | 1000 docs max |
| Batch create | 100 records max |
| Query timeout | 10 seconds |
| Populate fields | 5 per call |
| Populate records | 1000 per call |
| Collection name | `^[a-zA-Z][a-zA-Z0-9_]{0,63}$` |
| Field names | No `$` or `.` allowed |

- `appId` is server-set from OAuth token — apps cannot impersonate each other
- Schema validation via MongoDB `$jsonSchema` rejects malformed writes at the database level
- All queries go through a safe operator whitelist — no raw MongoDB operators exposed
- Soft-delete: `privos.db.delete` sets `_deletedAt`, records are excluded from queries automatically

### Full API Reference

See [Database Tools API Reference](./apis/tools-database.md) for all 16 tools with full input schemas and response formats.
