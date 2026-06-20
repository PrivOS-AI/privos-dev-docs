# App Platform — Security & Data Model

## Security

### Credential Push to App Server (Direct Apps)
- **Endpoint**: Hub POSTs `{ appId, clientId, clientSecret, timestamp }` to `{serverUrl}/.well-known/mcp/register` after a successful connect
- **Trust model**: registration push has **no signature** — relies on TLS for transport security and the developer's trust in the configured serverUrl
- **App-side defenses recommended**:
  - Enforce HTTPS for inbound (PrivOS only pushes over HTTPS in production)
  - Pin expected hub URL in app config (`EXPECTED_HUB_URL`) and reject if request does not originate from it
  - Treat the endpoint as idempotent (`appId` as upsert key) — Hub may retry future rotations
- **Best-effort**: failure (timeout, network error, non-2xx other than 404) does not abort the connect — admin can still wire credentials manually
- **404 = explicit skip**: apps that do not implement the endpoint return 404 and Hub logs nothing

### Direct Apps
- **Iframe sandbox**: deny-by-default — no `allow-same-origin` (prevents cookie/localStorage theft)
- **Permissions**: camera/microphone only granted if declared in `_meta.ui.permissions`
- **CSP**: app-declared CSP from `_meta.ui.csp` enforced
- **Scope enforcement**: every tool call checked against app's granted scopes
- **Room membership**: validated for room-scoped tools (messages, lists)
- **SSRF protection**: manifest fetch requires HTTPS in production, 5s timeout, 100KB max
- **PostMessage origin**: validated on message receipt
- **UI HTML proxy**: fetched server-side (no direct iframe src to external URL)
- **X-Frame-Options**: automatically exempted for `/api/v1/mcp-apps.ui-resource` route

### Relay Apps
- **OAuth bearer token**: WebSocket connection requires valid Bearer token in `Authorization` header
- **Token expiry**: tokens expire after 1 hour, app must refresh via `/oauth/token`
- **Pairing token expiry**: one-time pairing tokens expire after 1 hour if unused, persist after pairing
- **HTTPS-only wakeUrl**: if specified, only HTTPS URLs accepted for background wake calls
- **Cookie isolation**: `/apps/{appId}/ui` namespace prevents cookie leakage to other apps
- **Rate limiting**: per-app rate limits on relay WS connections and tool calls
- **Scope enforcement**: same as direct apps
- **Manifest fetch**: same SSRF protections as direct apps
- **App files in MinIO**: icons and assets stored at `.apps/{appId}/`, served via file management API

## Data Model

### mcp_apps collection

```typescript
{
  _id: string;
  appId: string;              // From manifest name (e.g., "com.example.app")
  name: string;               // Display title
  version: string;
  description: string;
  icon?: string;              // Relative path to icon in MinIO (e.g., "icon.png")
  iconFileId?: string;        // File ID in file management system (for serving via API)
  author: {
    name: string;
    email?: string;
    website?: string;
  };
  connectionType: 'direct' | 'relay';
  serverUrl?: string;         // MCP server base URL (optional for relay)
  relayUrl?: string;          // Relay endpoint URL (relay apps only)
  relayOnline?: boolean;      // Current relay connection status
  wakeUrl?: string;           // HTTPS endpoint to wake app (relay only)
  queueTtlMs?: number;        // Msg queue TTL when app offline (default: 3600000)
  queueMaxSize?: number;      // Max msgs in queue (default: 1000)
  tools: IMcpAppTool[];       // Discovered via tools/list
  scopes: string[];
  oauthAppId: string;         // Links to oauth_apps._id
  developerId: string;
  status: 'draft' | 'active' | 'suspended';
  installPermission: ('admin' | 'owner' | 'leader')[];
  entryPoints: {
    roomTab?: { toolName: string; title: string };
    sidebar?: { toolName: string; title: string };
    standalone?: { toolName: string; title: string };
  };
  localData?: Record<string, any>;
  _createdAt: Date;
  _updatedAt: Date;
}
```

### mcp_app_installations collection

```typescript
{
  _id: string;
  mcpAppId: string;
  appId: string;
  roomId?: string;        // null = workspace-level
  installedBy: string;
  installedAt: Date;
  settings?: Record<string, any>;
}
```

## File Structure

```
packages/
├── core-typings/src/
│   ├── IMcpApp.ts                           # IMcpApp + IMcpAppTool types
│   └── IMcpAppInstallation.ts               # Installation type
├── model-typings/src/models/
│   ├── IMcpAppsModel.ts                     # Model interface
│   └── IMcpAppInstallationsModel.ts         # Model interface
├── models/src/models/
│   ├── McpApps.ts                           # MongoDB model
│   └── McpAppInstallations.ts               # MongoDB model
├── app-react/                          # React hooks package
└── create-privos-mcp-app/                   # CLI scaffolder

apps/meteor/
├── server/
│   ├── oauth2-server/scope-definitions.ts   # Scope taxonomy
│   └── services/
│       ├── mcp-manifest-fetcher.ts          # Fetch /.well-known/mcp/manifest.json
│       ├── mcp-tool-discovery-client.ts     # MCP client initialize → tools/list
│       ├── mcp-ui-resource-fetcher.ts       # Fetch ui:// HTML
│       ├── mcp-rpc-dispatcher.ts            # Route runtime tool calls (direct HTTP vs relay WS)
│       ├── mcp-app-lifecycle-service.ts     # Connect/refresh/install/uninstall
│       ├── mcp-registration-pusher.ts       # POST credentials → /.well-known/mcp/register
│       ├── mcp-tool-registry.ts             # Tool definitions + scope enforcement
│       ├── mcp-tool-handlers-loader.ts      # Auto-imports all handlers
│       ├── mcp-tool-handlers-context.ts     # mcpapp.context.*
│       ├── mcp-tool-handlers-lists.ts       # mcpapp.lists.*
│       ├── mcp-tool-handlers-stages.ts      # mcpapp.stages.*
│       ├── mcp-tool-handlers-files.ts       # mcpapp.files.* + mcpapp.folders.*
│       ├── mcp-tool-handlers-messages.ts    # mcpapp.messages.*
│       ├── mcp-tool-handlers-bot.ts         # mcpapp.bot.* (token-authenticated bot actions)
│       ├── mcp-tool-handlers-rooms.ts       # mcpapp.rooms.*
│       ├── mcp-tool-handlers-users.ts       # mcpapp.users.*
│       ├── mcp-tool-handlers-db-schema.ts   # mcpapp.db.*Schema / *Collection
│       ├── mcp-tool-handlers-db-data.ts     # mcpapp.db.create/update/delete/get
│       ├── mcp-tool-handlers-db-query.ts    # mcpapp.db.query/count/aggregate/populate
│       ├── mcp-app-tools.ts                 # mcpapp.app.* (localData store)
│       ├── mcp-app-file-storage.ts          # MinIO storage for app icons/assets
│       ├── mcp-app-namespace-proxy.ts       # Per-app /apps/{appId}/* endpoints
│       ├── mcp-relay-pairing-token-store.ts # Generate/validate one-time pair tokens
│       ├── mcp-relay-connection-manager.ts  # Track relay online status + active sockets
│       ├── mcp-relay-websocket-endpoint.ts  # WS upgrade handler + auth check
│       └── mcp-relay-request-queue.ts       # Buffer messages while relay offline (TTL/cap)
├── app/api/server/v1/mcp-apps.ts           # REST endpoints (14 routes)
└── client/views/
    ├── admin/mcpApps/                      # Admin portal UI (incl. McpAppConnectForm)
    ├── room/mcp-apps/                      # Room tab/sidebar/host components
    └── mcp-apps/                           # Standalone page
```
