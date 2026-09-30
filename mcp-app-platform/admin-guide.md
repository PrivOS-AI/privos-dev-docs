# App Platform — Admin Guide

## Registering an App

### Direct App (via Admin Portal)

1. Navigate to **Admin → Apps** (`/admin/mcp-apps`)
2. Click **Connect App**
3. Select **Direct Connection**
4. Enter MCP server URL (e.g., `https://myapp.example.com`)
5. PrivOS performs two-step discovery:
   - **Step A**: Fetch `/.well-known/mcp/manifest.json` (name, version, author info)
   - **Step B**: MCP client connect → `initialize` → `tools/list`
6. OAuth credentials generated — **save clientId + clientSecret** (shown once)
7. Hub does a **best-effort POST** to `{serverUrl}/.well-known/mcp/register` with `{ appId, clientId, clientSecret, timestamp }`. If the app implements this endpoint it can auto-configure callback credentials. A 404 or network error is non-fatal — admin can still copy credentials manually from the response.

### Relay App (via Admin Portal — Auto-Pairing)

1. Navigate to **Admin → Apps** (`/admin/mcp-apps`)
2. Click **Add Standalone Relay App**. The Hub creates a one-time pairing URL at once (1-hour expiry):
   - Format: `wss://<hub>/api/v1/mcp-apps.relay?pair=<token>`
   - No manifest URL is entered: the app announces its own `privos-app.json` when it pairs.
3. **Share the pairing URL with the app developer**, and keep the **Hub fingerprint** for the check in step 5.
   - `POST /api/v1/mcp-apps.generate-pair-url` returns the URL and the fingerprint together. The admin screen shows the URL only; the fingerprint is also available from `GET /api/v1/mcp-apps.standalone.fingerprint`.
4. The developer runs `npm run pair` and pastes the URL. The app registers with the Hub and prints the fingerprint it received.
5. **Compare the two fingerprints over another channel** (a call or a separately verified chat), the same way you would accept an SSH host key. If they differ, do not approve.
6. The waiting screen polls until the app registers, then opens the approval dialog listing the permissions the app announced. Choose what to grant and approve. The developer's `npm run pair` has been waiting for this: it then receives its credentials, writes its identity file and starts.
7. After pairing you can tune the wake URL and the queue limits (see [Managing Relay Settings](#managing-relay-settings)).

How to keep the paired app running, update its manifest, and uninstall or re-pair it is in
[Install and operate your own MCP app](./install-and-operate-your-own-mcp-app.md).

### Via API

```bash
# Connect direct app
curl -X POST -H "Content-Type: application/json" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $UID" \
  -d '{"serverUrl": "https://myapp.example.com"}' \
  http://localhost:3000/api/v1/mcp-apps.connect

# Generate pairing URL for relay app
curl -X POST -H "Content-Type: application/json" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $UID" \
  -d '{}' \
  http://localhost:3000/api/v1/mcp-apps.generate-pair-url

# Response: { "pairUrl": "wss://<hub>/api/v1/mcp-apps.relay?pair=<token>", "pairToken": "<token>", "fingerprint": "<hub fingerprint>" }
# To pair an app that is already installed from its manifest, send {"mcpAppId": "<id>"} instead.

# Check pairing status (poll this endpoint to watch for app pairing)
curl -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $UID" \
  "http://localhost:3000/api/v1/mcp-apps.pair-status?token=<token>"

# Install in a room
curl -X POST -H "Content-Type: application/json" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $UID" \
  -d '{"mcpAppId": "<id>", "roomId": "<room_id>"}' \
  http://localhost:3000/api/v1/mcp-apps.install
```

## Admin Features

### Direct Apps
- **Connect App** — register MCP app by server URL
- **Settings** — configure who can install, permissions
- **Refresh** — re-fetch manifest and tools
- **Delete** — remove app, revoke tokens, cleanup

### Relay Apps
- **Generate Pairing URL** — create one-time URL for app developers to use during setup
- **Check Pairing Status** — monitor pairing progress with polling (waiting → paired → expired)
- **Settings** — configure wake URL, queue TTL/size, who can install
- **View Credentials** — show Client ID/Secret after pairing is complete
- **Check Online Status** — see if app is currently connected to relay
- **Delete** — remove app, revoke credentials, cleanup

## Managing Relay Settings

After a relay app is paired, admins can tune its queueing and wake behavior via `mcp-apps.updateSettings`. These control how the Hub handles messages sent while the app is offline.

| Setting | Type | Range | Default | Description |
|---------|------|-------|---------|-------------|
| `wakeUrl` | string (HTTPS only) | — | none | URL the Hub POSTs to when a queued message arrives and the relay is offline. Must start with `https://` — non-HTTPS rejected (SSRF protection) |
| `queueTtlMs` | number | 1000–120000 | 3600000 (1h) | TTL for messages buffered while app is offline. Clamped to the range on write |
| `queueMaxSize` | number | 1–100 | 1000 | Max buffered messages. Clamped to the range on write |
| `installPermission` | string[] | `admin`/`owner`/`leader` | `["admin"]` | Roles allowed to install. At least one required |
| `status` | string | `draft`/`active`/`suspended` | `active` | App-wide enable/disable |
| `uiUrl` | string | — | from manifest | Override the UI resource URL |

### Example — update queue + wake URL

```bash
curl -X POST -H "Content-Type: application/json" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $UID" \
  -d '{
    "_id": "app_abc123",
    "wakeUrl": "https://myapp.example.com/wake",
    "queueTtlMs": 60000,
    "queueMaxSize": 50
  }' \
  http://localhost:3000/api/v1/mcp-apps.updateSettings
```

### Example — restrict who can install

```bash
curl -X POST -H "Content-Type: application/json" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $UID" \
  -d '{
    "_id": "app_abc123",
    "installPermission": ["admin", "owner"]
  }' \
  http://localhost:3000/api/v1/mcp-apps.updateSettings
```

### Example — temporarily suspend

```bash
curl -X POST -H "Content-Type: application/json" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $UID" \
  -d '{"_id": "app_abc123", "status": "suspended"}' \
  http://localhost:3000/api/v1/mcp-apps.updateSettings
```

Out-of-range values for `queueTtlMs` and `queueMaxSize` are silently clamped to the documented bounds.

## Install Permissions

Configurable per app in Admin → Apps → Settings:

| Permission | Who can install |
|------------|----------------|
| `admin` | System admins (default) |
| `owner` | Room owners |
| `leader` | Room leaders |

## Installing from a Room

1. In a room, click the **layers icon** ("+" menu) in the tab bar
2. Click **"Install an App"** (below "+ New List")
3. A modal opens with a **search bar** and list of available apps
4. Each app shows: icon, name, description, author
5. Click **Install** to add the app to the room
6. Installed apps show **Open** (opens as room tab) and **Uninstall** buttons
7. Installed apps also appear in the folder menu under **"Installed Apps"** section

## Uninstalling a v3 BYO (standalone) App

A schemaVersion-3 BYO app — installed straight from its `privos-app.json`
without going through the marketplace — uses the **same one-button flow** as a
marketplace app: Settings → Uninstall → type UNINSTALL → Uninstall
permanently. There is no separate BYO uninstall path, and it is never blocked
by the BYO-install feature switch — disabling new installs never strands an
admin with an app they can't remove.

Uninstall tears down the runtime generation, room bindings, env config
(including encrypted secrets), the app's agent bot, and its relay OAuth client
and tokens. The app then **disappears from the admin list**, exactly as a
marketplace app does — the two acquisition modes look the same to an admin.

The catalog row is retained behind that (status `suspended`, manifest intact)
for audit and reinstall reuse, reachable only via
`GET /api/v1/mcp-apps.list?includeUninstalled=true`. Re-approving it provisions
a fresh generation and a new OAuth client, so the app must re-pair to pick up
new credentials. The admin apps list shows it when **Show uninstalled apps**
is enabled. Reinstalling means pairing the app server again.

`mcp-apps.delete` refuses an app that still has a live generation
(`error-active-generation-use-uninstall`) — use Settings → Uninstall instead.
Delete stays correct for a paired-but-never-approved app, an already-suspended
app, or a legacy non-v3 app.

Full install and operate detail: [Install and operate your own MCP app](./install-and-operate-your-own-mcp-app.md).
