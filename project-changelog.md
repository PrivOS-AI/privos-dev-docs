# PrivOS Chat - Project Changelog

## Overview

This document tracks significant features, improvements, and bug fixes released in PrivOS Chat. Changes are organized by date with associated commit references.

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
