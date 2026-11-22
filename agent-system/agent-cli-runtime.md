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

> **Out of date below:** since hub #780 a CLI login is installed and signed in to with Run command, and a pasted
> login is refused. The parts of this page that describe pasting a login (the login field of the room setting,
> "Getting the login", "a new paste") describe the earlier flow and still need rewriting.

## The room setting

Edit room, Sandbox settings, **Agent runtime**: Default, Claude Code CLI or Codex CLI, plus a field for the login.
It is saved through the room's sandbox configuration, so the same version check and push apply. The login is stored
encrypted, bound to the room's default bot only, and is never shown again; the owner sees "Saved", Replace and Clear.
The hub no longer asks for a typed sentence before saving or replacing; see
[Who may widen the exposure](#who-may-widen-the-exposure). Clearing is one click.

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

Only the agents recorded on the login: those in the VM when the login is saved, plus agents of the login's owner
that the owner adds to the team roster later (`joinTeamLogin` in `apps/meteor/server/services/a2a/a2a-teams.ts`). An
agent that qualifies for the VM later by other means (an admin reactivates it, an exclusion lifts) **stays in its own
VM**; the login is not dropped, and the card lists it as running outside the login until the owner adds it or saves
the login again. Nobody but the login's owner can put an agent on the login: the roster action answers "Only the
owner of the team CLI login can add this agent to the team", and such an agent that someone else adds to the team
room stays a plain team member. One of the owner's own agents counts as the owner here: when it invites the owner's
other agent into the team room, that agent is put on the login without a person acting (`rosterOnTeamJoin`,
`humanBehind`). Removing an agent from coverage or changing its owner takes it off the login without dropping it.

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

<!-- written from the design, check against the implementation before release -->
A team handover removes the login before its first step. Handing over a single agent takes that agent off the login
before its owner changes.
<!-- end of the part written from the design -->

## Who may widen the exposure

A login is readable by every member of every room its agents work in. The hub used to ask for a typed sentence before
an action that created or widened that exposure, and refused some paths outright (a bot adding a member, a room
created with a team agent in it, invite links, integrations). The sentence, its dialog and those refusals were removed
by hub #781. A login is now set up with Run command, which shows its own disclaimer before every run: the command has
full access to the sandbox, and anyone who may configure the sandbox can read what it stores, keys included.

One rule never depended on the sentence and came back with hub #786: an agent that runs on a team login is put into
a room by the owner of that login, and by nobody else.

What is left are permission rules:

| Action | Who may |
|---|---|
| Save, replace or clear a room's login | The room owner; a workspace admin may clear it ([The room setting](#the-room-setting)) |
| Save, replace or clear the team login | The owner of the team's covered agents; a workspace admin may clear it ([Agent teams](#agent-teams-one-login-for-the-team)) |
| Put an agent on the team login | The login's owner only, and only their own agents; an agent of the owner that invites one of them into the team room counts as the owner ([Which agents are on it](#which-agents-are-on-it)) |
| Add an agent that runs on a team login to a room | The owner of that login only (`apps/meteor/server/services/team-login-agent-add.ts`). **Refused** with `error-agent-cli-owner-only`, and nobody of that call is added: Members, an invite, `/invite`, the mention button, `teams.addMembers`, and a room, team or discussion created with the agent in its member list (a team created for a named owner is checked for the caller too). **Left out**, the others are added: a room made or created as a default room of a team the agent is in, "add all users", and the author of the message a discussion starts from. **Never by itself**: a channel join, the default channels, a trigger into a room it is not in, an item room; a bot is never the owner of a login. **Not asked**: the owner's super agent following them into their rooms, an agent re-attached to its own agent room, an agent that is already in the room, and an agent that creates a room |
| Add a person, or an agent that is on no team login, to a room | Whoever may add members to that room, on every path: Members, invites, joins, invite links, default channels, team default rooms, "add all users", integrations, the MCP rooms-invite tool, federation. No path checks whether a login can be read in the room |
| Add members as a bot | Through an invite (`channels.invite`, `groups.invite`, the `addUsersToRoom` method, `/invite`): only in the bot's own agent room, in a room it created, or in a room where its super-agent owner holds owner, admin or leader (`botMayInviteTo` in `apps/meteor/app/lib/server/methods/addUsersToRoom.ts`). `teams.addMembers`, a room created with a member list and a bot joining a public channel have no bot rule |

Every save, clear and removal of a login is a durable audit event of the `agent-team.cli-login-*` family in the hub's
server events (`-saved`, `-cleared`, `-drop-requested`, `-dropped`; `scope` says room or team). Rows written before
the removal may also carry `gate` and `ackVersion`, and the type `agent-team.cli-login-acknowledged`, which is no
longer written. The route and method parameters named `acknowledge` are still accepted and ignored, so older clients
keep working.

## Exposure and threat model

- **No consent step in the hub.** Nothing asks the person who adds a member to a room to confirm that the login
  becomes readable there. The owner is warned by the disclaimer Run command shows when the login is set up and by
  the texts on the team card and the room setting, and nothing stops a member of a room from reading the login.
  Which existing rooms a team-login agent is added to is its owner's decision: only they can add it
  ([Who may widen the exposure](#who-may-widen-the-exposure)). Memberships it had before the login was saved stay.
- **Who can drive an agent decides who can read its login.** A person who is not a member of the agent's own room
  can use the agent only in a room its owner opened ("Allow everyone using this Agent"), in the team room of a
  roster agent, or where it is the room's default bot (`apps/meteor/server/services/agent-mention-gate.ts`,
  `apps/meteor/app/api/server/v1/lib/agent-room-visibility.ts`). An agent in another room does not give that room's
  members the login. Adding a person to a room where the agent is open does, and a bot that may invite there (a super
  agent in a room where its owner holds authority) can do it without a person acting.
- **Every member of every room the team's agents work in can obtain the login**, through the agent or its files, and
  use the owner's subscription. A team VM gives no isolation between those rooms: files and vault relay tokens are
  reachable across projects. This is wider than the room login, which exposes one room.
- **Public channels.** An agent's output in a public channel is readable by people who are not members, through the
  workspace's public channel preview (permission `preview-c-room`, granted to the user and anonymous roles by default).
  Nothing in the hub covers that. Use **private** rooms for team agents.
- **Revocation is yours to finish.** Clearing removes the stored copy and the sandbox's copy. It does not revoke a copy
  someone already read. When you remove a member, or suspect a leak, revoke the token or session at the vendor and
  save a new login.
- **Vendor terms** apply as for the room login, and a login shared by a whole team is more likely to fall outside them.
  The owner is responsible.

