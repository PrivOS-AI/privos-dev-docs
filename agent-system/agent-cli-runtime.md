# Agent CLI Runtime

A room owner can run their own turns in a room on Claude Code or Codex, using their own vendor login, instead of the
default `privos-agent-sdk` runtime. Everyone else's turns in the same room keep using the default runtime. The CLI
runs inside the room's dedicated sandbox VM, next to the room's skills, vault bindings and MCP servers.

An agent team can also hold **one** login for all of its agents. In the team's shared VM every turn of those agents
runs on it, whoever asks. That is a different rule with a wider exposure; see
[Agent teams: one login for the team](#agent-teams-one-login-for-the-team).

> **Status:** off by default. A workspace admin turns on `PrivOSSandbox_Agent_Runtime_Enabled` (it also needs
> `PrivOSSandbox_Room_Config_Enabled`). The setting appears only when the room's sandbox can run the CLI; a tenant
> gets that at its next portal render, not at a release. Antigravity CLI exists in the code but is not offered until
> owners have a supported way to export its login (the CLI keeps it in the OS keyring).

## The room setting

Edit room, Sandbox settings, **Agent runtime**: Default, Claude Code CLI or Codex CLI, plus a field for the login.
It is saved through the room's sandbox configuration, so the same version check and push apply. The login is stored
encrypted, bound to the room's default bot only, and is never shown again; the owner sees "Saved", Replace and Clear.
Saving (or replacing) first asks the owner to type a fixed sentence acknowledging that every member of the room can
read and use the login; see [Typed acknowledgements](#typed-acknowledgements). Clearing is one click.

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

## Agent teams: one login for the team

An agent team with **Shared sandbox VM** on runs its agents in one team VM (see
[Shared sandbox VM](./bot-to-bot-protocol.md#shared-sandbox-vm)). The team owner can give that VM one Claude Code or
Codex login. It is set on the team's card under **Agents, Teams, Agent CLI login**, and uses the same paste formats as
the room setting ([Getting the login](#getting-the-login)).

**Conditions.** Both admin settings above are on; the team is enabled with Shared sandbox VM on and verified; at least
one agent runs in the VM; the team's sandbox offers team logins and the chosen CLI. The sandbox host is pinned when the
shared VM is verified, and the login is only ever sent there. Rules and reason codes:
`apps/meteor/server/services/a2a/a2a-team-cli-runtime.ts` in privos-hub; the sandbox side is in privos-sandbox
`docs/architecture/security.md` (Team cores).

| Who | Can |
|---|---|
| Owner of the team's agents | Set, replace, clear. Must own every agent that runs in the VM. |
| Workspace admin who is not the owner | Clear, and force-clear a removal the sandbox never confirmed. Not set or replace. |
| Everyone else | See the status at most. |

### Which turns run on it

In a team VM **every turn of an agent on the login runs on the CLI**: the owner's, any member's, an A2A wake from
another agent, a trigger. The room login's owner-turns-only rule does not apply here, because the owner decided that
the team shares one login rather than splitting turns by who asked. The one exception is an attempt that carries
execution resources (a bound skill): CLI runtimes cannot run those, so they stay on the default runtime and succeed
there instead of failing.

The model list of a team agent shows only the team CLI's models, in any room and to every member. Clearing the login
restores the normal list and resets stale picks.

Each project's own identity, room MCP servers, skill env and vault context still apply to its turns; the CLI child
never holds the VM's own key.

### Which agents are on it

Only the agents the owner acknowledged: those in the VM when the login is saved, plus agents added later with an
acknowledged roster add. An agent that qualifies for the VM later by other means (an admin reactivates it, an
exclusion lifts, it is rostered without the sentence) **stays in its own VM**; the login is not dropped, and the card
lists it as running outside the login until the owner adds it or saves the login again. Removing an agent from
coverage or changing its owner takes it off the login without dropping it.

### Removal

Clear, or any event below, removes the login. Turns of the team's agents are paused until the sandbox confirms it
removed the file, cancelled running CLI turns and emptied the team's CLI home; a paused turn is refused with a
"configuration pending" error after a short wait. Replacing a login pauses the same way. An admin can force-clear a
removal the sandbox cannot confirm; the file may then remain on the sandbox host until an operator purges it.

The login is removed automatically when: the admin settings are turned off; the team is turned off or its room is
deleted; Shared sandbox VM is turned off; no agent runs in the VM any more; the login's owner is deactivated or
deleted; the team's sandbox changed (including editing the sandbox URL or key of a covered agent room, which removes
the login on the old host first and is refused if that cannot be confirmed); the sandbox can no longer run the CLI; or
the team is handed over. Setting it again needs a new paste.

<!-- add-to-room gates: written from the design, check against the implementation before release -->
A team handover removes the login before its first step. Handing over a single agent takes that agent off the login
before its owner changes; the recipient is not asked for a sentence.
<!-- end add-to-room gates -->

## Typed acknowledgements

A login is readable by every member of every room its agents work in. Before an action that creates or widens that
exposure is accepted, the person doing it types one fixed English sentence, and the hub checks it on the server. The
exact wording is shown in the dialog and defined in `apps/meteor/lib/agent-cli-acknowledgement.ts`; it is compared
case-sensitively, ignoring surrounding spaces, in every language.

<!-- add-to-room gates: the rows "Add a team agent to a room", "Add a person to an exposed room" and the roster part of
"Team login" are written from the design, check against the implementation before release -->

| Gate | When | Who types it |
|---|---|---|
| Team login | Saving the team login, or adding an agent to the team roster while the team has a login. The dialog lists every room of the covered agents (direct messages and the team's default rooms included) with member counts, and the sandbox host. | The login's owner |
| Add a team agent to a room | Adding an agent on the login to an existing room. Covers the people added in the same action. Anyone but the owner is refused. | The login's owner |
| Add a person to an exposed room | Adding a person, or another owner's agent, to a room where an agent on a team login works or that holds a live room login; joining such a room is the same. | The person adding; the joiner for a join |
| Room login | Saving or replacing a room's login. Same words as adding a team agent to a room, recorded as its own gate. | The room owner |

Every accepted acknowledgement, and every save, clear and removal of a login, is a durable audit event of the
`agent-team.cli-login-*` family in the hub's server events (`scope` says room or team). The action is not accepted when
its row cannot be written. A refusal is logged but not stored as an event.

### Where a team agent can be added
<!-- add-to-room gates: written from the design, check against the implementation before release -->

While a team has a login, the rules below hold on every path that puts a member in a room, not only in the UI.

| Path | Result |
|---|---|
| Members, Add; the user-info action; `channels.invite`, `groups.invite` | The gate above, with the sentence sent as `acknowledge`. The hub answers `error-agent-cli-ack-required` and the client opens the dialog. |
| Join button and composer Join buttons of a room with an exposed agent | The joiner types the sentence. Sending a message does not join; the person is told to use the Join button. |
| Roster action (mark as team agent) | The owner types the team login sentence for an agent that would be covered. Another owner's agent needs the adder's room sentence. |
| Creating a room with a team agent among its members | Refused with `error-agent-cli-add-refused` (`create-with-agent`): create the room, then add the agent from Members. |
| Bot tokens (for example a super agent calling `channels.invite`), bot self-join | Refused (`bot-actor`). A human must add. |
| Incoming webhooks, `chat.postMessage` and outgoing triggers that would join their post-as user | Refused (`integration`), visible in the integration history. A member adds the user first. |
| Invite links and invites at registration | Refused (`invite-link`); the person asks a member to add them. |
| `/invite`, `/join`, "add them" from a mention | Refused with a hint to use Members, Add or the Join button. |
| Direct messages | Agent bots already refuse direct messages; unchanged. |
| Admin "add all users" to a room with a team agent | Refused up front (`no-modal`). |
| Team default rooms, auto-join toggle, default channels, item rooms, federation, trigger auto-invite into a room | A gated member or agent is skipped, not failed; the response or log says how many. |
| Team handover | The recipient add is exempt; the login is removed first (see Removal). |

<!-- end add-to-room gates -->

## Exposure and threat model

- **A consent record, not an access control.** The sentence is public and a script can send it. What the gate
  guarantees is that every accepted action carries the actor's explicit statement and an audit row naming who sent it.
  It does not stop a determined member of a room from reading the login.
- **Every member of every room the team's agents work in can obtain the login**, through the agent or its files, and
  use the owner's subscription. A team VM gives no isolation between those rooms: files and vault relay tokens are
  reachable across projects. This is wider than the room login, which exposes one room.
- **Public channels.** An agent's output in a public channel is readable by people who are not members, through the
  workspace's public channel preview (permission `preview-c-room`, granted to the user and anonymous roles by default).
  No gate covers that. Use **private** rooms for team agents.
- **Revocation is yours to finish.** Clearing removes the stored copy and the sandbox's copy. It does not revoke a copy
  someone already read. When you remove a member, or suspect a leak, revoke the token or session at the vendor and
  save a new login.
- **Vendor terms** apply as for the room login, and a login shared by a whole team is more likely to fall outside them.
  The owner is responsible.

