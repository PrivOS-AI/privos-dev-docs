# Agent CLI Runtime

A room owner can run their own turns in a room on Claude Code or Codex, using their own vendor login, instead of the
default `privos-agent-sdk` runtime. Everyone else's turns in the same room keep using the default runtime. The CLI
runs inside the room's dedicated sandbox VM, next to the room's skills, vault bindings and MCP servers.

> **Status:** off by default. A workspace admin turns on `PrivOSSandbox_Agent_Runtime_Enabled` (it also needs
> `PrivOSSandbox_Room_Config_Enabled`). The setting appears only when the room's sandbox can run the CLI; a tenant
> gets that at its next portal render, not at a release. Antigravity CLI exists in the code but is not offered until
> owners have a supported way to export its login (the CLI keeps it in the OS keyring).

## The room setting

Edit room, Sandbox settings, **Agent runtime**: Default, Claude Code CLI or Codex CLI, plus a field for the login.
It is saved through the room's sandbox configuration, so the same version check and push apply. The login is stored
encrypted, bound to the room's default bot only, and is never shown again; the owner sees "Saved", Replace and Clear.

| Who | Can |
|---|---|
| Room owner | Set, replace, clear |
| Workspace admin who is not the owner | Clear only (`runtime-owner-only` on set or replace) |
| Moderators, members | Nothing |

## Eligibility

All of these must hold, on save and on every later push. If one stops holding, the hub drops the runtime, deletes
the stored login and shows the reason in the setting.

- The admin settings above are on, and the sandbox advertises the provider (`agentCliRuntimes`).
- Sandbox collocation for the room is explicitly **Off** (not "Default"), so the bot has a dedicated VM.
- The room is not an agent room, it has a default bot, and that bot is not in an agent team.
- The saver is still an active owner of the room.

Rules and reason codes: `apps/meteor/server/services/privos-sandbox-agent-runtime.ts` in privos-hub. The sandbox
side is described in privos-sandbox `docs/architecture/agent-execution.md`.

## Whose turns run on the CLI

Only turns the owner starts: their chat messages, and agent triggers (cron, event, manual run) that the owner both
created and last edited. A trigger last edited by someone else runs as that person and stays on the default runtime.
Webhook-fired triggers never run on the CLI. A change of runtime on an existing thread goes through the normal
handoff summary and a fresh native session; it does not resume the other runtime's session.

Bound-skill attempts cannot run on a CLI runtime. If the CLI is missing from the core, the owner's turn fails with
`runtime-unavailable` instead of falling back silently.

## What the agent can see

The CLI sees the same room as the default runtime: project files, room files, the room's MCP servers, skill env and
vault-bound relay variables. The model list for the owner shows only that CLI's models; other members see the
normal list. Some names are withheld from a CLI because it reads them for its own login (for example
`OPENAI_API_KEY` and `OPENAI_BASE_URL` for Codex), so a skill that relies on a vault binding with exactly such a name
will not see it while that CLI runs. The CLI runs without its own approval prompts inside the VM.

## Any member can obtain the login

Routing the owner's turns to the CLI does not keep the login secret. Any member of the room can get the agent to
reveal it, through the room's agent or its files, and then use the owner's subscription. Set a runtime only in rooms
whose members you trust. When you remove a member, or suspect a leak, revoke the token or session at the vendor and
replace it in the setting; clearing the setting does not revoke a copy. The hub shows this as a hint in the setting.
This was accepted as a trade-off by the platform owner.

## Vendor terms

The login is a personal subscription. Whether it may be used this way, and by whom, is governed by the vendor's own
terms (Anthropic for Claude Code, OpenAI for Codex). The owner is responsible for complying with them.

## Getting the login

| CLI | What to paste | How |
|---|---|---|
| Claude Code | The token printed by `claude setup-token`, on the owner's own computer | One line |
| Codex | The contents of `~/.codex/auth.json` | Sign in with `codex login` on your own computer. The file exists only when Codex keeps its login in a file: set `cli_auth_credentials_store = "file"` in `~/.codex/config.toml` before signing in |

Signing in from inside the sandbox (device code) is not supported. A rejected login ends the owner's turn with an
error that points at this setting.
