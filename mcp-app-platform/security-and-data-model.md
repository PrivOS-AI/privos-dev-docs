# App Platform — Security & Data Model

## Security

- **Iframe sandbox**: deny-by-default — no `allow-same-origin` (prevents cookie/localStorage theft)
- **Permissions**: camera/microphone only granted if declared in `_meta.ui.permissions`
- **CSP**: app-declared CSP from `_meta.ui.csp` enforced
- **Scope enforcement**: every tool call checked against app's granted scopes
- **Room membership**: validated for room-scoped tools (messages, lists)
- **SSRF protection**: manifest fetch requires HTTPS in production, 5s timeout, 100KB max
- **PostMessage origin**: validated on message receipt
- **UI HTML proxy**: fetched server-side (no direct iframe src to external URL)
- **X-Frame-Options**: automatically exempted for `/api/v1/mcp-apps.ui-resource` route

## Data Model

### mcp_apps collection

```typescript
{
  _id: string;
  appId: string;          // From manifest name (e.g., "com.example.app")
  name: string;           // Display title
  version: string;
  description: string;
  icon?: string;          // Absolute URL to 96x96 PNG/SVG icon
  author: {
    name: string;
    email?: string;
    website?: string;
  };
  serverUrl: string;      // MCP server base URL
  tools: IMcpAppTool[];  // Discovered via tools/list
  scopes: string[];
  oauthAppId: string;     // Links to oauth_apps._id
  developerId: string;
  status: 'draft' | 'active' | 'suspended';
  installPermission: ('admin' | 'owner' | 'leader')[];
  entryPoints: {
    roomTab?: { toolName: string; title: string };
    sidebar?: { toolName: string; title: string };
    standalone?: { toolName: string; title: string };
  };
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
│       ├── mcp-manifest-fetcher.ts          # Fetch manifest metadata
│       ├── mcp-tool-discovery-client.ts     # MCP client initialize → tools/list
│       ├── mcp-ui-resource-fetcher.ts       # Fetch ui:// HTML
│       ├── mcp-app-lifecycle-service.ts     # Connect/refresh/install/uninstall
│       ├── mcp-tool-registry.ts             # Tool definitions + scope enforcement
│       ├── mcp-tool-handlers-*.ts           # Tool handlers (7 files)
│       └── mcp-tool-handlers-loader.ts      # Auto-imports all handlers
├── app/api/server/v1/mcp-apps.ts           # REST endpoints (11 routes)
└── client/views/
    ├── admin/mcpApps/                      # Admin portal UI
    ├── room/mcp-apps/                      # Room tab/sidebar/host components
    └── mcp-apps/                           # Standalone page
```
