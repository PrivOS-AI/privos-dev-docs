# App Platform — Admin Guide

## Registering an App

### Admin Portal

1. Navigate to **Admin → Apps** (`/admin/mcp-apps`)
2. Click **Connect App**
3. Enter MCP server URL (e.g., `https://myapp.example.com`)
4. Privos performs two-step discovery:
   - **Step A**: Fetch `/.well-known/mcp/manifest.json` (name, version, author info)
   - **Step B**: MCP client connect → `initialize` (with UI extension) → `tools/list`
5. OAuth credentials generated — **save clientId + clientSecret** (shown once)

### Via API

```bash
# Connect (register) an app
curl -X POST -H "Content-Type: application/json" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $UID" \
  -d '{"serverUrl": "https://myapp.example.com"}' \
  http://localhost:3000/api/v1/mcp-apps.connect

# Install in a room
curl -X POST -H "Content-Type: application/json" \
  -H "X-Auth-Token: $TOKEN" -H "X-User-Id: $UID" \
  -d '{"mcpAppId": "<id>", "roomId": "<room_id>"}' \
  http://localhost:3000/api/v1/mcp-apps.install
```

## Admin Features

- **Connect App** — register an MCP app by server URL
- **Settings** — configure who can install: System Admins, Room Owners, Room Leaders
- **Refresh** — re-fetch manifest metadata and tools
- **Delete** — remove app, revoke OAuth tokens, clean up installations

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
