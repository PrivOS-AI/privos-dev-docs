# PrivOS Chat - Project Changelog

## Overview

This document tracks significant features, improvements, and bug fixes released in PrivOS Chat. Changes are organized by date with associated commit references.

---

## 2026-04-29

### Agent Selector & Bot Key

#### Per-Agent Bot Key Validation
**Summary:** AI chat now re-validates the "Push bot key to Privos Brain" CTA per-agent. Switching agents in the selector triggers a fresh status fetch for the new bot, dismissals are scoped to `(roomId, botId)`, and pushes target the selected agent instead of always the room's default bot.

**Implementation:**
- `apps/meteor/app/api/server/v1/agent-privos-brain-bot-key.ts` — `agents.brain.botKeyStatus` (GET) and `agents.brain.pushBotKey` (POST) accept optional `botId`. `resolveTargetBotId` validates the bot is `type: 'bot'` and subscribed to the room before falling back to the default.
- `apps/meteor/server/services/privos-brain-bot-key-service.ts` — relaxed default-bot gate; now allows any bot subscribed to the room (membership-not-default-bot). Resolves `400 bot-not-room-default` when pushing for non-default agent bots.
- `apps/meteor/client/hooks/aiChat/useBotPrivosBrainKeyStatus.ts` — accepts `botId`, includes it in the React Query key and request params.
- `apps/meteor/client/components/AIChatBox/ChatBoxInput.tsx` — passes `selectedAgent.botUserId`; dismissal state keyed by `${roomId}:${botId}`.

#### Mid-Session Agent Switch — Full History Push
**Summary:** When the user changes agents during an active session, the next attempt force-pushes summary + activeMessages to the new agent's brain task. Previously, agent-room sessions used a botId-less `taskId` and fell into delta mode, leaving the new agent blind to prior conversation.

**Implementation:**
- `apps/meteor/app/agent-chat/server/contextCompaction.ts` — added `agentChanged = !!lastBotId && lastBotId !== botId`. Bypasses resume/delta short-circuits when an agent switch is detected. Per-bot scope (`${roomId}:${botId}`) and `lastBotId` updates after stream completion let resume mode kick in normally on subsequent same-agent turns.

#### Bot Self-Management Authorization
**Summary:** `agents.triggers.*` endpoints (list/add/update/remove/run) now accept calls from the bot itself when the bot has `owner` or `leader` role on its agent room. Skills running in Privos Brain authenticate with the bot's bearer token; previously the route required the *human* creator or admin permission, which broke autonomous trigger management.

**Implementation:**
- `apps/meteor/app/api/server/v1/agent-trigger-endpoints.ts` — `verifyBotOwnership` now also checks `Subscriptions.findOneByRoomIdAndUserId(agentRoomId, userId).roles` for `owner`/`leader` before falling back to admin permission. Scope is limited to this file's five trigger routes; other endpoints unchanged.

**Docs:** [Bot Key & Agent Switching](./agent-system/bot-key-and-agent-switching.md) — full lifecycle, failure modes, last-agent restoration logic.

---

## 2026-04-20

### Authorization

#### View Room Hidden Files Permission
**Summary:** New room-scoped permission `view-room-hidden-files` gates visibility of files and folders whose names start with `.` (e.g. `.env`, `.gitignore`). Default roles: `admin`, `owner`, `leader`. Members lose access to dot-prefixed files unless explicitly granted.

**Implementation:**
- `apps/meteor/app/authorization/server/constant/permissions.ts` — registers the permission (auto-loaded on startup; no migration required).
- `apps/meteor/app/api/server/v1/fileManagement.ts` — shared `HIDDEN_NAME_RE = /^\./` + `canSeeHidden(userId, roomId)` helper; enforced on:
  - List files (`GET /file-management.files.channel/:channelId`)
  - List folders (`GET /file-management.folders.channel/:channelId`) and `folders.all`
  - Folder content (`GET /file-management.folders/content/:fatherId`) — also blocks direct entry into a hidden folder
  - Root content (`GET /file-management.channels/:channelId/root`)
  - Stats (`GET /file-management.stats/:channelId`)
  - Single file / folder GET — returns `not found` to avoid leaking existence
  - Path resolve (`GET /file-management.resolve-path/:channelId`) — any hidden segment in the path is treated as unresolved
- `apps/meteor/server/models/Files.ts` & `Folders.ts` — `findFilesByChannel`, `countFilesByChannel`, `findFoldersByChannel`, `countFoldersByChannel`, `getFolderContent` accept optional `excludeHiddenRegex` that adds `{ name: { $not: regex } }` to the Mongo query (keeps paginated `total` consistent with filtered page).
- `packages/i18n/src/locales/en.i18n.json` — permission label + description.
- Tests: `apps/meteor/tests/unit/server/models/files-hidden-filter.tests.ts` — regex and query-shape coverage (7 passing).

**Behavior:** Recursion is intrinsic — every nested list call re-checks, so walking into a non-hidden folder still hides dot-prefixed children.

**Risk:** LOW — additive permission, default roles preserve existing UX for owners/leaders/admins.

---

## 2026-04-18

### UX Improvements

#### 1. Room Tab State Preservation
**Summary:** Switching between room tabs (Lists, Files, Documents, File Viewer, MCP Apps, Agent Settings) no longer unmounts the previous tab. Scroll position, form state, iframe contents, and in-memory data are preserved.

**Implementation:**
- `Room.tsx` caches every visited `mainContentTab` component in a `useRef<Map>` and toggles visibility via `display: contents | none` instead of swapping children on `createElement`.
- `McpAppTabWrapper.tsx` keeps every visited MCP app mounted per room session; only the active `mcpAppId` is visible. Iframe state and `postMessage` bridge survive tab switches.
- Messages view uses the same visibility-toggle pattern so returning from a tab does not re-render the message list.

**Trade-off:** Memory grows with the number of distinct tabs opened in a room; tabs are cleared when the room unmounts.

#### 2. "Open in New Window" for Rooms and Room Tabs
**Summary:** Rooms and room tabs can be opened in a standalone browser window.

**Features:**
- Sidebar (v1 + v2) `RoomMenu` kebab and `RoomContextMenu` right-click gain an **Open in new window** item when an `href` is provided by the caller (`SideBarItemTemplateWithData`, `TeamChannelItem`).
- Room tabs (`RoomTabs.tsx`) support **Shift+Click** and **right-click → Open in new window** on Messages, Files, Agent Settings, Lists, File Viewer, Documents, and MCP app tabs. The URL is computed via `router.buildRoutePath` preserving current route params and search.
- New window size matches current window dimensions with `noopener,noreferrer`.
- New i18n key: `Open_in_new_window`.

#### 3. MCP App Persistent Storage Proxy
**Summary:** MCP apps can now persist values across reloads via two new PostMessage bridge methods.

**Bridge methods (JSON-RPC 2.0):**
- `host/storage.get` → `{ key }` → `{ value }`
- `host/storage.set` → `{ key, value }` → `{ ok: true }`

**Security:** Values are stored in the host's `localStorage` under a fixed `mcp-app:` prefix. Apps cannot read/write host keys such as `Meteor.loginToken`. There is currently no per-app namespace — apps should self-prefix with their app id to avoid collisions.

**Related Documentation:**
- [Developer Guide — Host PostMessage Bridge](./mcp-app-platform/developer-guide.md#8-host-postmessage-bridge)

---

## 2026-04-16

### Completed Features

#### 1. Agent System — Conversational Agent Builder
**Summary:** Full agent creation flow powered by Privos Brain (Claude Code). Users chat with a builder assistant that gathers agent details, then the system provisions a bot user, private agent room, context files, and skill files.

**Features:**
- Conversational builder UI via `POST /v1/agents.builderChat` (Privos Brain-backed)
- Agent creation via `POST /v1/agents.create` — provisions bot user, room, context files, skill files
- Context files uploaded to MinIO: IDENTITY.md, MEMORY.md, CLAUDE.md
- Agent room auto-reply: `afterSaveMessage` → Privos Brain → streamed bot response
- Per-room concurrency guard (1 reply at a time)
- Context cache with 5-min TTL for MinIO file reads

**Related Documentation:**
- [Agent System Overview](./agent-system/index.md)
- [Agent Builder](./agent-system/agent-builder.md)
- [Agent Rooms](./agent-system/agent-rooms.md)
- [Architecture](./agent-system/architecture.md)

#### 2. Agent Trigger Registry
**Summary:** Unified automation system with 3 trigger types (cron, webhook, event), all funneling synthetic messages into agent rooms via `injectTriggerMessage()`. The existing reply handler processes them — no new AI pipeline needed.

**Features:**
- Heartbeat cron (every 60s) evaluates all cron triggers with atomic `lastRunAt` double-fire prevention
- Webhook receiver: `POST /v1/agents.webhook/:token` — public, token-based, optional HMAC verification
- Event triggers: `afterSaveMessage` dispatches `message.new` to subscribed agents with 30s cooldown
- 5 CRUD endpoints: `agents.triggers.list/add/update/remove/run`
- Max 5 triggers per agent, prompt max 500 chars
- Rate limiting: webhook 60 req/min per agent, event cooldown 30s per trigger
- Self-loop prevention for event triggers (skip agent's own room)
- Predefined intervals only (no raw cron expressions)

**Related Documentation:**
- [Trigger Registry](./agent-system/trigger-registry.md)
- [Trigger API Reference](./agent-system/trigger-api-reference.md)

#### 3. Agent Settings Tab (Room UI)
**Summary:** Dedicated room tab for agent rooms (alongside Chat, Files) housing trigger management UI with full CRUD.

**Features:**
- Pinned "Agent Settings" tab (cog icon) visible only in agent rooms
- Trigger list with type icons, schedule/event labels, prompt preview, last run timestamp
- Toggle switch to enable/disable triggers
- Add trigger form with type-conditional fields (schedule dropdown, event dropdown, prompt)
- Copy-to-clipboard for webhook URLs
- Max triggers callout when limit reached

**Related Documentation:**
- [Agent Settings UI](./agent-system/agent-settings-ui.md)

#### 4. Agent Self-Management Skills
**Summary:** Skill files uploaded to MinIO during agent creation, synced to Privos Brain CWD. Agents manage their own triggers via natural language → `tool_use` → curl API calls.

**Features:**
- 7 skill files: `.env`, `SKILL.md`, 5 JS scripts (list/add/update/remove/run)
- Scripts use `fetch()` with bot token auth against trigger API endpoints
- CLAUDE.md includes Skills section pointing to skill folder
- Natural language trigger management: "Set up a cron to check news every hour"

**Related Documentation:**
- [Self-Management Skills](./agent-system/self-management-skills.md)

#### 5. Room-Level Privos Brain Configuration
**Summary:** Per-room Brain settings (endpoint + API key + provider/model) in Edit Channel/Team Advanced Settings. When configured, AI Chat and Agent Bot replies route to room's Brain instead of global Privos Flow.

**Features:**
- Room customFields stores `privosBrain: { url, apiKey, defaultProvider?, defaultModel? }`
- `POST /v1/rooms.testBrainConnection` — validate Brain endpoint, fetch available providers/models
- AI Chat (`processAIRequestJob`) checks room Brain config before routing to Privos Flow
- Agent Bot (`handleAgentRoomMessage`) uses room Brain when available, falls back to global
- Edit Channel/Team → Advanced Settings → Privos Brain section with URL, API key, test connection, model selects
- Security: SSRF URL validation, apiKey stripped from all client-facing data paths, server-side apiKey preservation on partial updates

#### 6. Migration v348 — Agent Trigger Indexes
**Summary:** MongoDB indexes for agent trigger queries that run on hot paths (heartbeat cron every 60s, webhook token lookup).

**Indexes:**
- `agent_bot_type`: compound `{ type: 1, 'customFields.isAgentBot': 1 }` (sparse) — heartbeat cron + event dispatch
- `agent_trigger_webhook_token`: `{ 'customFields.agentTriggers.webhookToken': 1 }` (sparse) — webhook receiver

---

## 2026-04-13

### Completed Features

#### 1. MCP App Database Layer
**Summary:** Full app database capability via 16 new `privos.db.*` MCP tools with schema management, CRUD, typed queries, aggregation, and cascade references.

**Features:**
- Schema management: `registerCollection`, `updateSchema`, `getSchema`, `listCollections`, `dropCollection`
- CRUD operations: `create`, `createMany`, `get`, `update`, `updateMany`, `delete`, `deleteMany`
- Advanced queries: `query` (with filters, sort, pagination), `count`, `aggregate` (count/sum/avg/min/max)
- Reference resolution: `populate` (resolves 1-level references)
- Per-app collections in MongoDB with `$jsonSchema` validation
- Firestore-style safe query builder with operator whitelist
- Typed reference fields with cascade delete rules

**New OAuth Scopes:**
- `db:read` - Query and read records
- `db:write` - Create, update, delete records
- `db:schema:read` - Read collection schemas
- `db:schema:write` - Register/modify schemas

**Related Documentation:**
- [Tools — Database API](./mcp-app-platform/apis/tools-database.md) - Full database tool docs with examples
- [React SDK Reference](./mcp-app-platform/react-sdk-reference.md) - `useAppDb()` hook documentation

---

## 2026-03-26

### Completed Features

#### 1. Room-Scoped Agent Bot Management
**Commit:** `92db6b02` - feat(ai-chat): replace global agent list with room-scoped bot agents

**Summary:** Replaced global agent list UI with room-scoped bot agents. AI Chat now fetches agents specific to the current room instead of all public subagents from Privos Studio.

**Changes:**
- New endpoint: `GET /v1/rooms/:roomId/agentBots` - fetches bots configured for a specific room
- Renamed hook: `useAgents()` now accepts `roomId` parameter and fetches room-specific agents
- Removed: `GET /v1/agents.chat` endpoint (no longer used)
- UI shows "Default Agent" fallback when room has no agent bots configured
- Last-used agent per room is restored from localStorage with correct room context

**Impact:**
- Better isolation: agents only visible within their configured rooms
- Improved UX: agent dropdown reflects room-specific setup
- Performance: single API call per room instead of dual fetch

**Related Documentation:**
- [Room-Scoped Rooms API](./room-scoped-apis/rooms.md) - includes new agentBots endpoint

---

#### 2. Bot Configuration UI Redesign
**Commit:** `eb711818` - fix(bot-config): redesign agent flow selector with searchable dropdown

**Summary:** Replaced broken AutoComplete/SelectFiltered pattern with custom InputBox + Options + Chip pattern for better UX.

**Changes:**
- New agent flow source: `/v1/chatflows` filtered by `AGENTFLOW` type (replaces `/public-chatflows/bots/all`)
- Added `fetchAll` mode to `useAgents()` hook for bot management context
- Search by agent name or flow ID
- Removed `deployed` and `isPublic` gate on `agents.bots.getOne` endpoint
- Fixed `CreateBotModal` destructuring (agentBots → chatAgents)

**API Changes:**
- `GET /v1/chatflows?type=AGENTFLOW` - new source for agent flows
- `GET /v1/agents.bots/:botId` - removed deployed/isPublic restriction

**Related Documentation:**
- [useAgents Hook](./internal-apis/agent-chat.md) - hook signature and usage

---

#### 3. Admin Bot Management Page
**Commit:** `e7a44e99` - feat(admin): add Manage Bots page for system admins

**Summary:** New dedicated admin page allowing system admins to view, search, and manage all bots across users.

**Features:**
- Bot listing with search filter
- Bot owner display
- Edit and delete bot actions
- Sortable table columns
- Hubot sidebar icon in admin navigation

**New Routes:**
- `/admin/manage-bots` - main admin bot management page

**Impact:**
- Improved admin visibility of all bots
- Centralized bot management for admins
- Better bot lifecycle management

---

#### 4. File Version History & Restore
**Commit:** `2a82d5a7` - feat(files): add version history sidebar and restore functionality

**Summary:** Added file versioning with MinIO version listing and restore capability. Tracks editor metadata for each version.

**Features:**
- List all versions of a file stored in MinIO
- Editor metadata (userId, username, name) stored per version
- Restore previous file versions as current version
- FileVersionSidebar component integrated into FileMarkdownViewer

**New Endpoints:**
- `GET /api/v1/fileManagement/versions/:fileId` - list all versions with editor info
- `POST /api/v1/fileManagement/restore/:versionId` - restore a previous version

**API Changes:**
- File operations now store editor metadata in MinIO object metadata
- Version history accessible in file editor "more" menu

**Related Documentation:**
- [File Backups API](./internal-apis/file-backups.md) - updated with version endpoints

---

### Breaking Changes

None in this release.

---

### Deprecations

- `GET /v1/agents.chat` - Replaced by `GET /v1/rooms/:roomId/agentBots`
- `agents.chat.getOne` - No longer used in AI Chat context

---

### Dependencies & Prerequisites

- MinIO: Required for file versioning features
- Node.js: No version changes required
- MongoDB: No schema migrations required (uses existing Subscriptions and Users collections)

---

### Known Issues

None documented.

---

### Upgrade Notes

#### For End Users
- Agent dropdown in AI Chat now shows room-specific bots
- If no agents configured for a room, "Default Agent" will be used
- File version history is available in the file editor

#### For Administrators
- New "Manage Bots" page in Admin section for centralized bot management
- No configuration changes required for existing bot setups
- Agent visibility scoped to configured rooms

#### For Developers
- Agent fetching now requires `roomId` context
- API endpoints changed from `/agents.chat` to `/rooms/:roomId/agentBots`
- Hook signature: `useAgents({ roomId, enabled })` instead of `useAgents({ enabled })`

---

## Previous Releases

See commit history for earlier changes. Documentation of pre-2026-03-26 releases to be backfilled.
