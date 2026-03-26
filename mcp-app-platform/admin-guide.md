# App Platform — Admin Guide

## Registering an App

### Direct App (via Admin Portal)

1. Navigate to **Admin → Apps** (`/admin/mcp-apps`)
2. Click **Connect App**
3. Select **Direct Connection**
4. Enter MCP server URL (e.g., `https://myapp.example.com`)
5. Privos performs two-step discovery:
   - **Step A**: Fetch `/.well-known/mcp/manifest.json` (name, version, author info)
   - **Step B**: MCP client connect → `initialize` → `tools/list`
6. OAuth credentials generated — **save clientId + clientSecret** (shown once)

### Relay App (via Admin Portal — Auto-Pairing)

1. Navigate to **Admin → Apps** (`/admin/mcp-apps`)
2. Click **Generate Pairing URL**
3. Enter manifest URL (e.g., `https://myapp.example.com/.well-known/mcp/manifest.json`)
4. Optionally configure:
   - **Wake URL**: HTTPS endpoint Privos calls to wake app (if offline)
   - **Queue TTL**: How long to queue messages while app is offline (default: 1 hour)
   - **Queue Max Size**: Max messages to buffer (default: 1000)
5. Privos generates a pairing URL (1-hour expiry):
   - Format: `https://chat.privos.com/pair?token=pair_abc_123xyz`
6. **Share the pairing URL with the app developer**
7. Developer enters URL during `npm start` → credentials auto-saved to .env
8. Waiting UI shows status with polling until app pairs
9. On pairing complete, admin can view credentials (clientId, clientSecret) in app settings

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
  -d '{
    "manifestUrl": "https://myapp.example.com/.well-known/mcp/manifest.json",
    "wakeUrl": "https://myapp.example.com/wake",
    "queueTtlMs": 3600000,
    "queueMaxSize": 1000
  }' \
  http://localhost:3000/api/v1/mcp-apps.generate-pair-url

# Response: { "pairUrl": "https://...", "pairToken": "pair_abc_123xyz", "expiresIn": 3600 }

# Check pairing status (poll this endpoint to watch for app pairing)
curl -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $UID" \
  "http://localhost:3000/api/v1/mcp-apps.pair-status?token=pair_abc_123xyz"

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
