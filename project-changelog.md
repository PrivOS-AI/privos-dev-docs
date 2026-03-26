# PrivOS Chat - Project Changelog

## Overview

This document tracks significant features, improvements, and bug fixes released in PrivOS Chat. Changes are organized by date with associated commit references.

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
