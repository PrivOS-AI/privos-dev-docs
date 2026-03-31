# PrivOS Documentation — Super Detailed Structure Outline

**Date:** 2026-03-30 | **Purpose:** Blueprint for writing all PrivOS feature docs
**Status:** OUTLINE ONLY — content to be written per section

---

## Directory Tree

```
docs/
├── README.md                                          # Docs home — index of all sections with links
│
├── getting-started/
│   ├── what-is-privos.md                              # Product overview, philosophy, pillars
│   ├── key-concepts-and-terminology.md                # Glossary: rooms, lists, items, stages, bots, agents, MCP, etc.
│   ├── quickstart-self-hosted-deployment.md            # 10-min deploy (PM2, Docker), env vars, first login
│   └── platform-architecture-overview.md              # High-level system diagram, tech stack, data flow
│
├── platform-concepts/
│   ├── rooms-as-workspaces.md                         # Room = workspace unit, what lives inside a room
│   ├── room-scoped-data-isolation.md                  # How data is isolated per room, security model
│   ├── user-roles-and-permissions.md                  # Admin, owner, moderator, member, guest — RBAC
│   ├── data-sovereignty-and-self-hosting.md            # Why self-hosted matters, GDPR/HIPAA/SOC2 implications
│   └── cross-room-operations.md                       # Cross-room assignment, shared folders, global search
│
├── team-communication/
│   ├── channels-and-direct-messages.md                # Public/private channels, DMs, multi-party DMs
│   ├── threads-and-discussions.md                     # Threaded replies, discussion rooms, following threads
│   ├── messaging-features.md                          # Reactions, mentions, pinned messages, starred, search
│   ├── voice-and-video-calls.md                       # WebRTC calls, Jitsi integration, screen sharing
│   ├── file-sharing-in-chat.md                        # Inline file uploads, image previews, audio messages
│   ├── notifications-and-preferences.md               # Push, email, desktop notifications, per-channel settings
│   ├── message-actions-and-formatting.md              # Markdown, code blocks, slash commands, message actions
│   └── mobile-and-desktop-apps.md                     # Supported platforms, push notifications, offline support
│
├── lists/
│   ├── lists-overview.md                              # What lists are, versatile structured-data containers
│   │   ### Sections:
│   │   # - Definition: room-embedded mini-databases
│   │   # - Use case matrix (CRM, tickets, inventory, HR, etc.)
│   │   # - List vs traditional project boards
│   │   # - Key capabilities summary
│   │
│   ├── kanban-board-view.md                           # Drag-and-drop cards, stage columns, card layout
│   │   ### Sections:
│   │   # - Stage columns and drag-drop
│   │   # - Card display (title, assignee, due date, custom fields)
│   │   # - Filtering and sorting in kanban
│   │   # - Stage WIP limits (if applicable)
│   │   # - Empty state and onboarding
│   │
│   ├── spreadsheet-table-view.md                      # Inline cell editing, column resize, row operations
│   │   ### Sections:
│   │   # - Table layout and column configuration
│   │   # - Inline cell editing (click-to-edit)
│   │   # - Column types and rendering per field type
│   │   # - Row selection, multi-select
│   │   # - Column sorting (single + multi-column, clear sort)
│   │   # - Column resize and reorder
│   │   # - Keyboard navigation (Tab, Enter, Escape, Arrow keys)
│   │
│   ├── custom-fields.md                               # Field types, configuration, color-coded options
│   │   ### Sections:
│   │   # - Supported field types: TEXT, SELECT, MULTI_SELECT, DATE, NUMBER
│   │   # - Creating and editing custom fields
│   │   # - Color-coded select options (palette)
│   │   # - Field ordering and visibility
│   │   # - Field-level validation rules
│   │   # - Default values
│   │
│   ├── workflow-stages.md                             # Stage pipeline, customization, drag reorder
│   │   ### Sections:
│   │   # - What stages are (pipeline columns)
│   │   # - Creating, renaming, reordering, deleting stages
│   │   # - Stage colors and icons
│   │   # - Default stage assignment for new items
│   │   # - Stage-based filtering
│   │   # - Webhook events on stage change
│   │
│   ├── items-and-sub-items.md                         # Item CRUD, sub-item hierarchy, item detail view
│   │   ### Sections:
│   │   # - Creating items (quick add, full form)
│   │   # - Item detail panel (title, description, assignees, fields)
│   │   # - Sub-items: creating, nesting, collapsing
│   │   # - Moving items between stages (drag, dropdown, API)
│   │   # - Assigning users to items (in-room, cross-room)
│   │   # - Due dates and date-based alerts
│   │   # - Item search and filtering
│   │
│   ├── saved-views.md                                 # Custom filter/sort/view configs, per-user persistence
│   │   ### Sections:
│   │   # - What saved views are
│   │   # - Creating a view (filter + sort + view type + visible columns)
│   │   # - Switching between views
│   │   # - Editing and deleting views
│   │   # - Default view per list
│   │   # - Per-user vs shared views
│   │
│   ├── xlsx-export-import.md                          # Export to XLSX, import with diff preview
│   │   ### Sections:
│   │   # - Export: what's included (items, fields, stages)
│   │   # - Export: dual-sheet format (data + metadata)
│   │   # - Import: uploading XLSX file
│   │   # - Import: diff preview modal (added, modified, deleted items)
│   │   # - Import: conflict resolution
│   │   # - Import: field type mapping and validation
│   │   # - Round-trip fidelity (export → edit in Excel → import back)
│   │
│   ├── clipboard-and-spreadsheet-operations.md        # Copy/paste cells, multi-cell selection, bulk operations
│   │   ### Sections:
│   │   # - Selecting cells and ranges
│   │   # - Copy (Ctrl+C) — single cell, multi-cell, row
│   │   # - Paste (Ctrl+V) — into cells, creating new rows
│   │   # - Cut operations
│   │   # - Tab-separated value format compatibility
│   │   # - Paste from Excel/Google Sheets
│   │
│   ├── undo-redo.md                                   # Operation history stack, supported operations
│   │   ### Sections:
│   │   # - Supported undoable operations (cell edit, move, delete, create)
│   │   # - Undo (Ctrl+Z) and Redo (Ctrl+Y) keyboard shortcuts
│   │   # - History stack depth and persistence
│   │   # - Multi-step undo
│   │
│   ├── list-settings-and-access-control.md            # Per-list permissions, list configuration
│   │   ### Sections:
│   │   # - List creation and ownership
│   │   # - List-level access control (who can view, edit, manage)
│   │   # - List settings panel (name, description, default view)
│   │   # - Archiving and deleting lists
│   │
│   └── incremental-backup-and-restore.md              # Time Machine for lists, full + delta snapshots
│       ### Sections:
│       # - What incremental backup is (full + delta snapshots)
│       # - MinIO storage for backup data
│       # - Manual backup trigger
│       # - Automatic backup scheduling
│       # - Restore from specific point in time
│       # - Backup chain integrity and validation
│       # - Admin configuration (retention, frequency)
│       # - Restore preview before applying
│
├── file-management/                                   # [EXISTING — extend]
│   ├── file-management-features.md                    # [EXISTS] General features overview
│   ├── file-management-api.md                         # [EXISTS] API endpoints
│   ├── minio-storage-and-architecture.md              # MinIO setup, bucket structure, presigned URLs
│   │   ### Sections:
│   │   # - MinIO as S3-compatible object store
│   │   # - Bucket naming and room-scoped paths
│   │   # - Presigned URL generation for downloads
│   │   # - Timestamp-prefixed filenames for uniqueness
│   │   # - Connection pooling and configuration
│   │
│   ├── file-versioning-and-restore.md                 # Version history, editor metadata, restore previous
│   │   ### Sections:
│   │   # - How versioning works (MinIO object versioning)
│   │   # - Version list API endpoint
│   │   # - Editor metadata per version (userId, username, name)
│   │   # - Restoring a previous version
│   │   # - Version diff (if applicable)
│   │   # - Storage impact of versioning
│   │
│   ├── folder-hierarchy-and-navigation.md             # Unlimited depth folders, move, rename, breadcrumbs
│   │   ### Sections:
│   │   # - Creating folders and nested folders
│   │   # - Moving files and folders (drag-drop, context menu)
│   │   # - Renaming with duplicate detection
│   │   # - Breadcrumb navigation
│   │   # - Folder-level operations (delete, archive)
│   │
│   ├── chunked-upload-and-large-files.md              # Files over 100MB, multipart upload, progress tracking
│   │   ### Sections:
│   │   # - FileUpload_MaxDirectFileSize threshold (default 100MB)
│   │   # - Multipart upload flow (initiate → parts → complete)
│   │   # - Progress tracking and resumable uploads
│   │   # - File type validation and size limits
│   │
│   ├── markdown-editor.md                             # WYSIWYG editing, Mermaid diagrams, export
│   │   ### Sections:
│   │   # - Rich text editing (bold, italic, headings, lists, tables)
│   │   # - Mermaid diagram support (flowcharts, sequence, gantt)
│   │   # - Code blocks with syntax highlighting
│   │   # - Image embedding
│   │   # - Export to PDF, HTML, DOCX
│   │   # - Auto-save and version creation on save
│   │
│   ├── code-file-editor.md                            # Syntax highlighting, language detection, in-app editing
│   │   ### Sections:
│   │   # - Supported languages and syntax highlighting
│   │   # - Language auto-detection by file extension
│   │   # - Editing and saving code files
│   │   # - Line numbers, word wrap, theme
│   │
│   ├── ai-auto-parse.md                               # Automatic file-to-markdown conversion for AI analysis
│   │   ### Sections:
│   │   # - What AI Auto Parse does
│   │   # - Supported file formats (PDF, DOCX, images, etc.)
│   │   # - Parsing pipeline (upload → extract → store markdown)
│   │   # - How AI agents consume parsed content
│   │   # - Configuration and toggling
│   │
│   ├── cross-room-shared-folders.md                   # Share file directories across rooms/teams
│   │   ### Sections:
│   │   # - Creating shared folders
│   │   # - Linking shared folders to multiple rooms
│   │   # - Permission model for shared folders
│   │   # - Sync and conflict handling
│   │   # - Removing shared folder access
│   │
│   ├── file-search-and-filters.md                     # Search by name, type, date, user, size, content
│   │   ### Sections:
│   │   # - Text search in file names
│   │   # - Filter by file type (document, image, video, code)
│   │   # - Filter by date range, uploader, file size
│   │   # - Full-text search in parsed content (if AI parsed)
│   │   # - Sort options (name, date, size, type)
│   │
│   └── file-backup-and-time-machine.md                # Backup and restore for files, admin config
│       ### Sections:
│       # - Time Machine concept for files
│       # - Full backup and incremental delta snapshots
│       # - Restore flow (select point in time → preview → apply)
│       # - Admin configuration (backup frequency, retention)
│       # - Storage consumption and cleanup
│
├── ai-agents/
│   ├── ai-agents-overview.md                          # What AI agents are, how they work in PrivOS
│   │   ### Sections:
│   │   # - Definition: room-scoped, context-aware AI assistants
│   │   # - How agents differ from ChatGPT/Copilot/Gemini
│   │   # - Sense → Think → Act cycle
│   │   # - Agent capabilities (read lists, files, conversations, take action)
│   │   # - Sandboxing philosophy
│   │
│   ├── room-scoped-ai-isolation.md                    # Each room has its own agent, data boundaries
│   │   ### Sections:
│   │   # - One agent per room model
│   │   # - What data the agent can see (room files, lists, messages)
│   │   # - What data the agent CANNOT see (other rooms)
│   │   # - Cross-room agent restrictions
│   │   # - Compliance implications (GDPR, HIPAA data segregation)
│   │
│   ├── agent-sandboxing-and-security.md               # Permission boundaries, audit trail, rate limits
│   │   ### Sections:
│   │   # - Permission boundaries (read, write, delete per resource)
│   │   # - No external data leakage (self-hosted processing)
│   │   # - Audit trail: every agent action logged
│   │   # - Rate limiting and resource caps
│   │   # - Human-in-the-loop gates (approval buttons)
│   │   # - What happens when agent is misconfigured
│   │
│   ├── context-aware-sessions.md                      # AI chat sessions, context resume, session history
│   │   ### Sections:
│   │   # - Session model: conversation history per room
│   │   # - Context injection: files, list data, room metadata
│   │   # - Session resume on reconnect
│   │   # - Canvas artifacts (structured outputs, tables, code)
│   │   # - Session token management (JWT-based)
│   │   # - Multi-agent session switching
│   │
│   ├── agent-configuration-and-management.md          # Setting up agents, selecting AI models, admin controls
│   │   ### Sections:
│   │   # - Creating agent bots for a room
│   │   # - Selecting agent flow (from PrivOS Studio)
│   │   # - Agent bot management page (admin)
│   │   # - Default agent fallback behavior
│   │   # - Agent visibility and room assignment
│   │   # - Agent deletion and cleanup
│   │
│   ├── privos-studio-integration.md                   # No-code AI flow builder, drag-and-drop workflows
│   │   ### Sections:
│   │   # - What PrivOS Studio is (external AI flow builder)
│   │   # - Agent flows vs chat flows
│   │   # - Connecting Studio flows to room agents
│   │   # - Flow types: AGENTFLOW, CHATFLOW
│   │   # - Studio API endpoints (/v1/chatflows)
│   │   # - Building custom AI workflows (overview, link to Studio docs)
│   │
│   ├── interactive-decision-buttons.md                # Inline action buttons, user interaction model
│   │   ### Sections:
│   │   # - What interactive buttons are
│   │   # - Button types (action, confirm, select, URL)
│   │   # - Inline keyboards in chat messages
│   │   # - Button callback flow (user taps → agent receives → processes)
│   │   # - Designing button layouts
│   │   # - Use cases: approval workflows, quick actions, escalation
│   │
│   └── autonomous-workflow-examples.md                # Real-world scenarios: sales, support, HR, inventory
│       ### Sections:
│       # - Self-driving sales pipeline
│       # - Autonomous customer support triage
│       # - HR onboarding automation
│       # - Inventory monitoring and reorder
│       # - Weekly executive report generation
│       # - Custom autonomous workflow patterns
│
├── bot-api/                                           # [EXISTING top-level files → reorganize into folder]
│   ├── bot-api-overview.md                            # What the Bot API is, Telegram-style design philosophy
│   │   ### Sections:
│   │   # - Bot API concept: bots as workspace automation actors
│   │   # - Telegram Bot API inspiration and similarity
│   │   # - Bot vs Agent vs MCP App (when to use what)
│   │   # - Capabilities summary
│   │
│   ├── bot-token-authentication.md                    # Token format, permissions, scopes, lifecycle
│   │   ### Sections:
│   │   # - Token format: privos_<userId>_<secret>
│   │   # - Auto-generated 64-char hex secret
│   │   # - Permissions: read, write, admin
│   │   # - Scopes: messages:read, messages:write, lists:read, lists:write, etc.
│   │   # - Token expiration and deactivation
│   │   # - Token rotation and security best practices
│   │
│   ├── message-operations.md                          # Send, edit, delete messages as bot
│   │   ### Sections:
│   │   # - Send message (text, markdown, attachments)
│   │   # - Edit message
│   │   # - Delete message
│   │   # - Send with inline keyboard (action buttons)
│   │   # - Send media (images, files, audio)
│   │   # - Message formatting options
│   │
│   ├── webhook-system.md                              # Event subscriptions, delivery, retry, signatures
│   │   ### Sections:
│   │   # - Webhook registration and URL configuration
│   │   # - Event filtering by type and room
│   │   # - HMAC-SHA256 signature verification
│   │   # - Retry logic with exponential backoff (up to 24 attempts)
│   │   # - Redis/BullMQ queue for reliable delivery
│   │   # - Webhook delivery status and debugging
│   │
│   ├── supported-webhook-events.md                    # [EXISTS in bot-webhooks/] All 18 event types detailed
│   │   ### Sections:
│   │   # - Message events (sent, updated, deleted)
│   │   # - List events (created, updated, deleted)
│   │   # - Item events (created, updated, deleted, stage changed)
│   │   # - Stage events (created, updated, deleted)
│   │   # - File events (uploaded, updated, deleted)
│   │   # - Folder events (created, updated, deleted)
│   │   # - Member events (joined, left)
│   │   # - Room events
│   │   # - Full payload schema per event type
│   │
│   ├── webhook-payload-examples.md                    # [EXISTS in bot-webhooks/] Request/response examples
│   │
│   ├── bot-management-admin.md                        # Admin page, create/edit/delete bots, room assignment
│   │   ### Sections:
│   │   # - Creating a bot (UI flow)
│   │   # - Bot owner and permissions
│   │   # - Assigning bots to rooms
│   │   # - Admin manage bots page (/admin/manage-bots)
│   │   # - Bot lifecycle: create → configure → deploy → monitor → delete
│   │
│   └── webhook-queue-architecture.md                  # [EXISTS] BullMQ queue, Redis, delivery guarantees
│
├── mcp-app-platform/                                  # [EXISTING — extend]
│   ├── overview.md                                    # [EXISTS] Architecture, key concepts
│   ├── developer-guide.md                             # [EXISTS] Scaffold, build, manifest
│   ├── api-reference.md                               # [EXISTS] REST endpoints, MCP tools, scopes
│   ├── react-sdk-reference.md                         # [EXISTS] @privos/app-react hooks
│   ├── admin-guide.md                                 # [EXISTS] Register, configure, install apps
│   ├── security-and-data-model.md                     # [EXISTS] Sandbox, scopes, collections
│   ├── connection-modes.md                            # Direct HTTP vs Relay WebSocket, when to use which
│   │   ### Sections:
│   │   # - Direct HTTP mode: architecture, requirements, setup
│   │   # - Relay WebSocket mode: architecture, NAT traversal, setup
│   │   # - Comparison table (latency, requirements, security)
│   │   # - Pairing token flow (one-time-use, 1-hour expiry)
│   │   # - Reconnection and keep-alive
│   │   # - When to use which mode
│   │
│   ├── iframe-sandboxing-deep-dive.md                 # Browser sandbox restrictions, PostMessage bridge
│   │   ### Sections:
│   │   # - Default deny-all sandbox policy
│   │   # - Allowed permissions: allow-scripts only
│   │   # - Conditional permissions: camera, microphone (manifest-declared)
│   │   # - What sandboxed apps CANNOT do (steal sessions, redirect, XSS)
│   │   # - Server-side HTML proxy (no direct external URL loading)
│   │   # - PostMessage JSON-RPC 2.0 bridge protocol
│   │   # - Origin validation on every message
│   │
│   ├── oauth-scopes-and-permissions.md                # All available scopes, runtime enforcement
│   │   ### Sections:
│   │   # - Scope definitions (lists:read, lists:write, messages:read, etc.)
│   │   # - Scope request in app manifest
│   │   # - Admin approval workflow for scopes
│   │   # - Runtime scope enforcement in MCP Tool Registry
│   │   # - Scope escalation prevention
│   │
│   ├── room-installation-and-lifecycle.md             # Installing apps in rooms, app tabs, uninstall
│   │   ### Sections:
│   │   # - Installing an app into a room
│   │   # - App tab rendering in room UI
│   │   # - Room membership validation for app access
│   │   # - Uninstalling apps from rooms
│   │   # - App update and version management
│   │
│   └── building-industry-apps-examples.md             # Concrete examples: sales dashboard, HR, IoT, etc.
│       ### Sections:
│       # - Example: Lead Scoring Dashboard (Sales)
│       # - Example: Campaign Analytics (Marketing)
│       # - Example: Employee Onboarding (HR)
│       # - Example: Invoice Tracker (Finance)
│       # - Example: IoT Dashboard (Operations)
│       # - Example: Contract Review (Legal)
│       # - App architecture pattern for each
│
├── security-and-compliance/
│   ├── security-overview.md                           # Security philosophy, defense-in-depth layers
│   │   ### Sections:
│   │   # - Self-hosted security model
│   │   # - Defense-in-depth: network → auth → room isolation → resource-level
│   │   # - Zero-trust principles
│   │   # - Security vs SaaS competitors (data control comparison)
│   │
│   ├── room-level-data-isolation.md                   # How rooms enforce data boundaries at every layer
│   │   ### Sections:
│   │   # - Room as security boundary
│   │   # - Database-level isolation (queries scoped by roomId)
│   │   # - API-level enforcement (middleware checks)
│   │   # - AI agent isolation (agent sees only its room)
│   │   # - Bot isolation (webhooks filtered by room membership)
│   │   # - MCP app isolation (room installation validation)
│   │   # - File storage isolation (room-prefixed paths in MinIO)
│   │
│   ├── authentication-and-authorization.md            # Login methods, RBAC, API auth (tokens, JWT, API keys)
│   │   ### Sections:
│   │   # - User authentication methods (password, LDAP, SAML, OAuth)
│   │   # - Two-factor authentication (2FA/TOTP)
│   │   # - Role-based access control (RBAC)
│   │   # - API authentication: user tokens, bot tokens, internal API keys
│   │   # - Room session JWT tokens (room-scoped APIs)
│   │   # - Token expiration, refresh, and revocation
│   │
│   ├── data-sovereignty-and-compliance.md             # GDPR, HIPAA, SOC2, data residency
│   │   ### Sections:
│   │   # - Self-hosted = full data control
│   │   # - GDPR compliance: data processing, right to erasure, DPA
│   │   # - HIPAA considerations: PHI handling, audit logs, access control
│   │   # - SOC2 alignment: security, availability, confidentiality
│   │   # - Data residency: deploy in any jurisdiction
│   │   # - No vendor data mining, no third-party access
│   │   # - Data export and portability
│   │
│   ├── audit-logging.md                               # What's logged, log format, retention, querying
│   │   ### Sections:
│   │   # - User action audit logs
│   │   # - AI agent action logs (read, write, send, notify)
│   │   # - Bot action logs
│   │   # - MCP app action logs
│   │   # - Admin action logs
│   │   # - Log format and storage
│   │   # - Log retention policies
│   │   # - Querying and exporting logs
│   │
│   ├── webhook-security.md                            # HMAC signatures, HTTPS enforcement, SSRF protection
│   │   ### Sections:
│   │   # - HMAC-SHA256 payload signing
│   │   # - Signature verification on receiver side
│   │   # - HTTPS enforcement for webhook URLs
│   │   # - SSRF protection (timeout, size limits, URL validation)
│   │   # - IP allowlisting (if applicable)
│   │
│   └── rate-limiting-and-abuse-prevention.md          # Rate limits per API, per app, per bot, per user
│       ### Sections:
│       # - Rate limit tiers (standard, batch, bot, MCP app)
│       # - Rate limit headers in responses
│       # - Per-IP, per-user, per-bot, per-app limits
│       # - Abuse detection and auto-blocking
│       # - Resource caps for AI agents
│       # - Configuring rate limits
│
├── administration/
│   ├── deployment-guide.md                            # PM2, Docker, production setup, env vars
│   │   ### Sections:
│   │   # - Prerequisites (Node.js, MongoDB, Redis, MinIO)
│   │   # - Development mode deployment (deploy-pm2-dev.sh)
│   │   # - Production mode deployment (deploy-pm2.sh)
│   │   # - Docker deployment (if applicable)
│   │   # - Environment variables reference (.env)
│   │   # - PM2 management commands
│   │   # - Reverse proxy setup (Nginx/Caddy)
│   │   # - SSL/TLS configuration
│   │
│   ├── environment-variables-reference.md             # Complete .env reference with all variables
│   │   ### Sections:
│   │   # - Application settings (ROOT_URL, PORT)
│   │   # - MongoDB connection (MONGO_URL, MONGO_OPLOG_URL)
│   │   # - Redis configuration (REDIS_URL)
│   │   # - MinIO configuration (endpoint, credentials, bucket)
│   │   # - AI/LLM provider settings
│   │   # - PrivOS Studio/Flow URL
│   │   # - Internal API key
│   │   # - JWT secrets
│   │   # - Email/SMTP settings
│   │   # - Feature flags
│   │
│   ├── minio-setup-and-management.md                  # Installing MinIO, bucket policies, monitoring
│   │   ### Sections:
│   │   # - MinIO installation (standalone, distributed)
│   │   # - Bucket creation and policies
│   │   # - Versioning enablement
│   │   # - Access credentials and IAM
│   │   # - Monitoring and health checks
│   │   # - Backup and disaster recovery
│   │
│   ├── mongodb-configuration.md                       # Replica set, oplog, separate file management DB
│   │   ### Sections:
│   │   # - Replica set requirement for oplog
│   │   # - Main database vs file management database (separate)
│   │   # - Connection string format
│   │   # - Index optimization
│   │   # - Backup strategies
│   │
│   ├── redis-configuration.md                         # Redis for caching, sessions, job queues
│   │   ### Sections:
│   │   # - Redis role in PrivOS (caching, JWT tokens, job queues)
│   │   # - Connection configuration
│   │   # - BullMQ queue setup
│   │   # - Memory management
│   │
│   ├── admin-panel-overview.md                        # Admin UI features, navigation, settings
│   │   ### Sections:
│   │   # - Accessing admin panel
│   │   # - User management
│   │   # - Room management
│   │   # - Bot management (/admin/manage-bots)
│   │   # - MCP app management
│   │   # - File backup configuration
│   │   # - General settings
│   │
│   ├── backup-and-disaster-recovery.md                # Full system backup strategy, restore procedures
│   │   ### Sections:
│   │   # - MongoDB backup (mongodump, automated)
│   │   # - MinIO backup (mc mirror, replication)
│   │   # - Redis backup (RDB snapshots)
│   │   # - List incremental backups (built-in time machine)
│   │   # - File version history (built-in)
│   │   # - Full system restore procedure
│   │   # - Disaster recovery playbook
│   │
│   └── monitoring-and-logging.md                      # PM2 monitoring, log management, health checks
│       ### Sections:
│       # - PM2 monitoring (pm2 monit, pm2 status)
│       # - Application logs (pm2 logs)
│       # - Structured logging format
│       # - Health check endpoints
│       # - Performance monitoring
│       # - Alerting setup
│
├── api-reference/
│   ├── api-overview.md                                # API types, authentication methods, common patterns
│   │   ### Sections:
│   │   # - Three API types: REST API, Internal API, Room-Scoped API
│   │   # - Bot API (separate authentication)
│   │   # - MCP App API (OAuth scopes)
│   │   # - Common response format
│   │   # - Pagination, filtering, sorting conventions
│   │   # - Error codes reference
│   │
│   ├── rest-api/
│   │   └── README.md                                  # Standard Rocket.Chat REST API extensions
│   │       ### Sections:
│   │       # - Authentication (user token + userId)
│   │       # - PrivOS-specific endpoints (lists, items, stages, files, etc.)
│   │       # - Endpoint naming conventions
│   │
│   ├── internal-apis/                                 # [EXISTING — keep as-is, extend if needed]
│   │   ├── README.md                                  # [EXISTS] Overview, auth, rate limits
│   │   ├── lists.md                                   # [EXISTS]
│   │   ├── items.md                                   # [EXISTS]
│   │   ├── stages.md                                  # [EXISTS]
│   │   ├── documents.md                               # [EXISTS]
│   │   ├── rooms.md                                   # [EXISTS]
│   │   ├── users.md                                   # [EXISTS]
│   │   ├── channels.md                                # [EXISTS]
│   │   ├── groups.md                                  # [EXISTS]
│   │   ├── ai-messages.md                             # [EXISTS]
│   │   ├── agent-chat.md                              # [EXISTS]
│   │   ├── agent-chat-session.md                      # [EXISTS]
│   │   ├── shared-folders.md                          # [EXISTS]
│   │   └── file-backups.md                            # [EXISTS]
│   │
│   └── room-scoped-apis/                              # [EXISTING — keep as-is, extend if needed]
│       ├── README.md                                  # [EXISTS] Overview, auth flow, JWT
│       ├── lists.md                                   # [EXISTS]
│       ├── items.md                                   # [EXISTS]
│       ├── stages.md                                  # [EXISTS]
│       ├── documents.md                               # [EXISTS]
│       ├── rooms.md                                   # [EXISTS]
│       ├── files.md                                   # [EXISTS]
│       └── ai-chat-sessions.md                        # [EXISTS]
│
├── developer-guide/
│   ├── developer-overview.md                          # Developer entry point, what you can build
│   │   ### Sections:
│   │   # - Three extension points: Bot API, MCP Apps, Internal APIs
│   │   # - When to use Bot API vs MCP App vs direct API
│   │   # - Developer environment setup
│   │   # - Contributing to PrivOS
│   │
│   ├── building-bots-quickstart.md                    # Step-by-step: create bot, get token, handle webhooks
│   │   ### Sections:
│   │   # - Create a bot user
│   │   # - Generate bot token
│   │   # - Register webhook URL
│   │   # - Handle incoming events (Node.js example)
│   │   # - Send messages back (with inline keyboards)
│   │   # - Deploy your bot
│   │
│   ├── building-mcp-apps-quickstart.md                # Step-by-step: scaffold, develop, register, install
│   │   ### Sections:
│   │   # - npx create-privos-mcp-app my-app
│   │   # - Project structure walkthrough
│   │   # - MCP server setup
│   │   # - React UI development
│   │   # - Manifest configuration
│   │   # - Register app with PrivOS admin
│   │   # - Install in a room
│   │   # - Using @privos/app-react SDK hooks
│   │
│   ├── integration-patterns.md                        # Common patterns: event-driven, polling, bidirectional
│   │   ### Sections:
│   │   # - Event-driven (webhook → process → respond)
│   │   # - Polling (periodically check lists/items)
│   │   # - Bidirectional sync (external system ↔ PrivOS list)
│   │   # - AI-powered automation (event → AI analysis → action)
│   │   # - Multi-bot coordination
│   │
│   └── api-client-examples.md                         # Code examples in Node.js, Python, cURL
│       ### Sections:
│       # - Node.js/TypeScript examples
│       # - Python examples
│       # - cURL examples
│       # - Authentication setup per language
│       # - Error handling patterns
│
├── use-cases/
│   ├── sales-pipeline-crm.md                          # How to use PrivOS as a CRM
│   │   ### Sections:
│   │   # - Setting up a Sales room
│   │   # - Creating a pipeline list with stages (Lead → Qualified → Proposal → Closed)
│   │   # - Custom fields for deals (Company, Deal Size, Contact)
│   │   # - AI agent for lead scoring and follow-up reminders
│   │   # - Bot webhook for external lead capture
│   │   # - Weekly pipeline reports via AI agent
│   │
│   ├── customer-support-ticketing.md                  # Ticket system with auto-triage
│   │   ### Sections:
│   │   # - Support room setup
│   │   # - Ticket list with priority/category fields
│   │   # - AI agent for auto-classification and assignment
│   │   # - Bot for external ticket intake (email, form, chat widget)
│   │   # - SLA tracking with date fields
│   │   # - Resolution workflow
│   │
│   ├── project-management.md                          # Task boards, sprint tracking
│   │   ### Sections:
│   │   # - Project room with Kanban board
│   │   # - Sprint planning with stages
│   │   # - Sub-items for task breakdown
│   │   # - Cross-team assignment
│   │   # - Progress tracking with AI reports
│   │
│   ├── hr-recruitment-onboarding.md                   # Hiring pipeline, onboarding checklists
│   │   ### Sections:
│   │   # - Recruitment pipeline list
│   │   # - Candidate tracking with custom fields
│   │   # - Onboarding checklist as sub-items
│   │   # - Automated onboarding via AI agent
│   │   # - Document storage in room files
│   │
│   ├── inventory-and-operations.md                    # Product catalog, stock tracking, reorder alerts
│   │   ### Sections:
│   │   # - Inventory list with SKU, price, stock fields
│   │   # - Low-stock alerts via AI agent
│   │   # - Reorder workflow with action buttons
│   │   # - Supplier management
│   │   # - XLSX export for reporting
│   │
│   └── knowledge-base-and-documentation.md            # File management as internal wiki/docs
│       ### Sections:
│       # - Organizing knowledge in folders
│       # - Markdown files as wiki pages
│       # - AI Auto Parse for searchable content
│       # - Version history for document governance
│       # - Shared folders for cross-team access
│
├── project-changelog.md                               # [EXISTS] — continues to be maintained
│
└── glossary.md                                        # Complete keyword/term reference
    ### Terms to include:
    # - Room
    # - Channel (public/private)
    # - Direct Message (DM)
    # - Thread
    # - List
    # - Item
    # - Sub-item
    # - Stage (workflow stage)
    # - Custom Field
    # - Saved View
    # - Kanban Board
    # - Spreadsheet View
    # - Bot
    # - Bot Token
    # - Webhook
    # - Webhook Event
    # - Inline Keyboard
    # - AI Agent
    # - Agent Flow
    # - PrivOS Studio
    # - Context-Aware Session
    # - Canvas Artifact
    # - MCP (Model Context Protocol)
    # - MCP App
    # - Direct HTTP (connection mode)
    # - Relay WebSocket (connection mode)
    # - App Manifest
    # - OAuth Scope
    # - Sandboxed Iframe
    # - PostMessage Bridge
    # - Room Session Key
    # - Internal API
    # - Room-Scoped API
    # - MinIO
    # - Presigned URL
    # - Chunked Upload
    # - AI Auto Parse
    # - Shared Folder
    # - File Versioning
    # - Time Machine (backup/restore)
    # - Incremental Backup
    # - Full Snapshot
    # - Delta Snapshot
    # - XLSX Export/Import
    # - Diff Preview
    # - Clipboard Operations
    # - Undo/Redo
    # - RBAC (Role-Based Access Control)
    # - Data Sovereignty
    # - HMAC Signature
    # - BullMQ
    # - Pairing Token
```

---

## File Count Summary

| Section | New Files | Existing Files | Total |
|---|---|---|---|
| **getting-started/** | 4 | 0 | 4 |
| **platform-concepts/** | 5 | 0 | 5 |
| **team-communication/** | 8 | 0 | 8 |
| **lists/** | 12 | 0 | 12 |
| **file-management/** | 8 | 2 | 10 |
| **ai-agents/** | 8 | 0 | 8 |
| **bot-api/** | 8 | 3 | 11 |
| **mcp-app-platform/** | 4 | 6 | 10 |
| **security-and-compliance/** | 7 | 0 | 7 |
| **administration/** | 8 | 0 | 8 |
| **api-reference/** | 2 | 15 | 17 |
| **developer-guide/** | 5 | 0 | 5 |
| **use-cases/** | 6 | 0 | 6 |
| **Root files** | 2 | 1 | 3 |
| **TOTAL** | **87** | **27** | **114** |

---

## Migration Notes (Existing Files)

| Current Location | Action | New Location |
|---|---|---|
| `docs/BOT_API_FEATURE.md` | Merge content into | `docs/bot-api/bot-api-overview.md` + sub-files |
| `docs/BOT_WEBHOOK_API.md` | Merge content into | `docs/bot-api/webhook-system.md` |
| `docs/bot-webhooks/index.md` | Merge into | `docs/bot-api/webhook-system.md` |
| `docs/bot-webhooks/supported-webhook-events-reference.md` | Move to | `docs/bot-api/supported-webhook-events.md` |
| `docs/bot-webhooks/webhook-payload-examples.md` | Move to | `docs/bot-api/webhook-payload-examples.md` |
| `docs/WEBHOOK_QUEUE_ARCHITECTURE.md` | Move to | `docs/bot-api/webhook-queue-architecture.md` |
| `docs/MCP_APP_PLATFORM.md` | Keep as index | Update links to new sub-files |
| `docs/file-management/*` | Keep | Extend with new files |
| `docs/internal-apis/*` | Move to | `docs/api-reference/internal-apis/` |
| `docs/room-scoped-apis/*` | Move to | `docs/api-reference/room-scoped-apis/` |
| `docs/mcp-app-platform/*` | Keep | Extend with new files |
| `docs/project-changelog.md` | Keep at root | No change |

---

## Writing Priority

| Priority | Section | Reason |
|---|---|---|
| **P0** | getting-started/ | First thing anyone reads |
| **P0** | lists/ | Core differentiator, most complex feature |
| **P0** | ai-agents/ | Key selling point, no existing docs |
| **P1** | security-and-compliance/ | Enterprise buyers need this |
| **P1** | use-cases/ | Sales enablement, demos |
| **P1** | platform-concepts/ | Foundation for understanding everything |
| **P2** | team-communication/ | Mostly inherited from Rocket.Chat, less unique |
| **P2** | bot-api/ | Partially documented, needs reorganization |
| **P2** | developer-guide/ | Important for ecosystem growth |
| **P3** | administration/ | Ops teams need this, but deploy guide already exists |
| **P3** | mcp-app-platform/ | Already well-documented, just extend |
| **P3** | api-reference/ | Already exists, reorganize |
| **P3** | file-management/ | Partially exists, extend |
