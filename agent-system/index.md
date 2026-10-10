# Agent System

AI agents in PrivOS Hub — conversational bots powered by PrivOS Sandbox (Claude Code) with automated triggers, self-management skills, and a dedicated room-based workspace.

## Quick Links

- [Architecture](./architecture.md) — System overview, data flow, component map
- [Agent Builder](./agent-builder.md) — Conversational agent creation flow
- [Agent Rooms](./agent-rooms.md) — Room structure, auto-reply handler, context files
- [Trigger Registry](./trigger-registry.md) — Cron, webhook, and event triggers
- [Trigger API Reference](./trigger-api-reference.md) — REST endpoints for trigger CRUD + webhook receiver
- [Agent Settings UI](./agent-settings-ui.md) — Room tab for managing triggers
- [Self-Management Skills](./self-management-skills.md) — Skill files agents use to manage their own triggers
- [Bot Key & Agent Switching](./bot-key-and-agent-switching.md) — Bot-key push to PrivOS Sandbox, agent selector re-validation, mid-session context handover
- [Agent Harness Runtime](./agent-harness-runtime.md) — Second runtime: an operator-owned ACP bridge (Claude Code / Codex / Cursor / Goose / custom) instead of PrivOS Sandbox — protocol, dispatch switch, relay lifecycle, pairing, per-room isolation
- [Credential Vault](./credential-vault.md) — Egress-injected secrets for external APIs: shared, room and agent-private bindings, secret requests, coexistence with the bot key
- [Room Services](./room-services.md) — Supervised processes that outlive an agent's turn (listeners, long jobs, pollers): declaration, states, owner stop, raw (board) and relay (VM) credentials, limits, the `privos-services` skill
- [Bot-to-Bot Protocol](./bot-to-bot-protocol.md) — `agents.a2a.*`: roster-scoped messages between agents with hub-enforced rules, chains, caps, approvals, the `privos-team` skill and the `privos agents a2a` CLI
- [Super Agent](./super-agent.md) — A trusted agent bot that mirrors its owner's rooms and roles, writes app-owned list items, listens through hub-side subscription filters, replies in the owner's DMs after approval, and reaches the hub from its VM through the proxy egress

## Concepts

| Concept | Description |
|---------|-------------|
| **Agent** | A bot user with `customFields.isAgentBot: true`, backed by PrivOS Sandbox |
| **Agent Room** | Private room (`agent-room-{botId}`) where the agent lives and responds |
| **PrivOS Sandbox** | Claude Code instance that processes agent messages via HTTP API |
| **Trigger** | Automation rule (cron/webhook/event) stored in `customFields.agentTriggers[]` on bot user |
| **Synthetic Message** | Message with `t: 'agent-trigger'` injected into agent room by trigger system |
| **Context Files** | IDENTITY.md, CLAUDE.md, MEMORY.md uploaded to MinIO, synced to PrivOS Sandbox CWD |
| **Skill Files** | JS scripts in `.claude/skills/privos-agent-management/` that agents execute via `tool_use` |

## How It Works (30-second overview)

1. User creates agent via **Agent Builder** (conversational UI or manual form)
2. System creates bot user + private agent room + uploads context/skill files to MinIO
3. When user sends message in agent room → **reply handler** forwards to PrivOS Sandbox → streams response back
4. **Triggers** (cron/webhook/event) inject synthetic messages into agent room → same reply handler processes them
5. Agents can **self-manage** triggers via skill scripts (natural language → `tool_use` → curl API calls)
