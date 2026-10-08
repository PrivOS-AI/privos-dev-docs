# Super Agent

A **super agent** is an agent bot that a workspace admin has marked as a trusted extension of its owner. The PrivOS Hub
keeps the bot in every channel and private group its owner is in (with the owner's room role), lets it write items of
MCP-app-owned lists, evaluates subscription filters on its event triggers, and lets it answer the owner's DMs in the
owner's name after the owner approves. From its VM the agent reaches rooms, lists and items with the `privos` CLI through
the sandbox proxy egress, so the bot key never enters the VM.

Nothing happens until an admin sets the flag on a bot. Operator steps (enable, disable, emergency stop, rollback) live in
the ops runbook, not here.

## Why this shape

- **Delegation by permission, not identity.** An agent never holds a human's token. The credential vault refuses a
  binding for the workspace hub host on purpose, so "give the agent a hub PAT" is not an option. A super agent acts under
  its own identity, and what it may do is derived, at request time, from what its owner may do in the same room.
- **Bot authority never exceeds the owner's.** Membership and room role mirror the owner. Where the owner is a plain
  member the bot is one.
- **Only the owner can start its turns.** Everything the bot reads (messages, list items, notifications) is untrusted
  content; limiting who can start a turn is the main containment.
- **Filters are data, not code.** The hub evaluates declarative filters. It never runs user code.
- **Everything that widens authority is audited durably, and one switch freezes it** (see [Kill switch](#kill-switch)).

## Flag and eligibility

Routes (source: `apps/meteor/app/api/server/v1/bots-super-agent.ts`):

| Route | Who | Purpose |
|---|---|---|
| `POST /v1/bots.superAgent.set` `{ botId, enabled, resetDeclined? }` | holders of permission `manage-super-agents` (role `admin` by default) | Set or clear the flag. `resetDeclined` is `'all'` or a list of room ids the mirror may join again |
| `GET /v1/bots.superAgent.get?botId=` | the owner, or a `manage-super-agents` holder; never the bot itself | The state document (flag, owner, room records) |
| `POST /v1/bots.superAgent.dmReply.set` `{ botId, mode }` | the owner, or a `manage-super-agents` holder; never the bot | Switch DM replies between `approve` and `auto` |

Enabling is refused unless the bot is an active ordinary agent bot (not the Universal Assistant, not an installation bot)
whose creator (`_createdBy`) is an active person. That creator becomes the **owner** (`principalId`) and is frozen into the
state at enable time. Clearing skips every eligibility lookup.

State lives in its own collection, never in `customFields`: persona and user saves replace `customFields` whole, and the
mirror needs per-room records that survive them. Schema: `apps/meteor/server/core-typings/ISuperAgentState.ts`.

Clearing the flag leaves every room the mirror joined, restores the roles the bot held before the mirror touched them,
removes the extra catalog routes, and keeps the state document and the `declined` records. Setting the flag again reuses
the records.

## Owner-only turns

A super agent answers its owner and nobody else. One check (`isSuperAgentTurnAllowed`) sits at every place an agent turn
starts, for both runtimes (sandbox and harness): the mention wrapper (which also covers the agent-room post handler), the
streaming-room loop, the AI Chat window (`ai-messages.send` and `uploadAttachment`), `agents.triggers.run`, the room
visibility check, and the harness `respondTo` rule, which is forced to `owner`. A super agent never posts the "not
enabled" notice to anyone. These checks read the flag only, not the kill switch, so a frozen bot still obeys its owner
alone.

A super agent's **agent room** (`agent-room-<botId>`) should hold exactly one person, its owner. Turns that carry the
owner's DMs or notifications are delivered only while that holds (see [Filters](#subscription-filters)).

## Two keys for app-owned item writes

Lists owned by an MCP app normally accept item writes only from the app. A super agent may create, update, move and delete
**items** of such lists through `internal/rooms/:rid/items*` when **both** hold, checked on every request:

1. the flag is set, the kill switch is released and the owner is an active person;
2. the room is a mirrored, non-declined room, the owner holds `owner`, `admin` or `leader` there **now**, and the bot holds
   a role the mirror aligned. A role someone hands the bot by hand carries no authority and is realigned by the mirror.

Each such write leaves one `super-agent.item-write` audit row **before** the write runs (a failing audit write blocks the
write). The row is written only where the app-owned rule would have refused the caller; ordinary lists leave no row.
Batch create and batch move write one row per request, batch update and delete one per item.

Structure stays app-only: fields, stages and list deletion keep today's rule, and app objects outside lists stay closed
(the app's own tools are the way in). Isolated lists are untouched. Apps cannot opt out; the audit row is the control.

Source: `assertAppOwnedItemWrite` in `apps/meteor/app/api/server/helpers/list-room-access.ts`, authority in
`apps/meteor/server/services/super-agent/super-agent-authority.ts`. The item routes are listed in
[room-scoped items](../room-scoped-apis/items.md).

## Room mirror

The hub keeps the bot in every **channel and private group** the owner is in, with the owner's `owner`, `admin` or `leader`
role (owner is mirrored as owner; moderator is ignored). Joining posts one system message, authored by the admin who set
the flag, naming the bot and the owner.

**Never mirrored:** direct messages, livechat and voice rooms, any agent room (including the bot's own), personal inbox
rooms, list-item rooms, discussions, archived rooms, rooms declined (below), and everything when the owner is not an
active person. The qualifying rule is `roomQualifiesForDefaultBot` plus the archived and agent-room checks in
`apps/meteor/server/services/super-agent/super-agent-mirror.ts`.

**Agent team rooms** are mirrored like any other channel: plain membership with the owner's role. The join passes no
inviter, so the A2A roster hook does not fire and the mirror never enrols the bot in a roster. Joining an agent team stays a
separate, explicit act.

**When it runs.** A pass runs immediately when the flag changes and within seconds of the owner joining, leaving, being
removed, creating a room, having a room role changed through the role methods, or being deactivated or reactivated. A cron
(`SuperAgent_Reconcile_Interval_Minutes`, default 5, read at startup) catches everything else, such as a role change made
any other way. One pass runs at a time per bot (lease on the
state document); a pass is slow and sequential on purpose.

**Declined rooms.** If a person removes the bot from a room the mirror could join, that room is marked `declined` and is not
re-added. An admin can allow it again with `resetDeclined`. The mirror's own leaves never mark a room declined.

**Rooms gained through a team join** (a team main room pulls in the team's default rooms) are recorded as mirror joins and
left again on clear. Rooms the bot was already in are kept on clear; only the roles it held before are restored.

**Last owner.** The bot can co-own a room, so the last-owner guards (`leaveRoom`, `removeUserFromRoom`, the deactivation
handover) count **people** only when a person is the one leaving: the owner cannot leave or be removed while the bot is the
only other owner. Deactivation hands a person's rooms to a human before a bot. The mirror itself never leaves or demotes
in a room where that would leave no owner; it stays and records `last-owner`.

## Room binding modes

Bot-key calls carry the room their session is bound to. `Bot_Bearer_Room_Binding_Mode` (settings group `Agent_Delegation`)
is `audit` (default: log a mismatch) or `enforce` (refuse it). A super agent works in both modes. Under `enforce` an active
super agent may additionally address its own agent room and every mirrored, non-declined room (by path, query or, for the
allowlisted room-management writes, the JSON body `roomId`/`rid`), and call `rooms.get`, `subscriptions.get` and
`subscriptions.getOne`. A room addressed by name stays refused. Message-addressed reads that check the room after loading
the message (`chat.*`) keep the plain `enforce` rule.

Source: `bot-bearer-route-policy.ts` and `super-agent-room-binding.ts` under `apps/meteor`, wired in `ApiClass.ts`.

## Subscription filters

An **event trigger** (`agents.triggers.add` with `type: 'event'`) may carry a `filter` instead of a single `event`. The hub
evaluates the filter before it injects a turn into the agent room, so the agent wakes only for events that matter.
Endpoint fields: [Trigger API Reference](./trigger-api-reference.md).

| Field | Meaning |
|---|---|
| `events` | Required, non-empty: `message.new`, `notification.created` |
| `subject` | `bot` (default: mentions, replies and DMs about the agent) or `principal` (about its owner; super agents only) |
| `rooms` | `'mirrored'` (the mirrored rooms; super agents only) or an explicit list of room ids. Default for `principal` is `mirrored`; for `bot` every room the bot is in |
| `includeDms` | `principal` only, default `true`: also evaluate the owner's one-to-one DMs (the bot does not join them) |
| `match` | Any-of predicates (below). An empty `match` matches every event in scope |
| `coalesce` | `{ windowSeconds, maxEvents }`: hold events for the window and deliver them as one turn |
| `cooldownSeconds` | Minimum gap between turns of this trigger, default 30, `0` allowed |

`match` predicates, true if **any** holds: `mention` (the subject is mentioned), `reply` (a reply in a thread the subject
posted in, or a quote of the subject), `hotRoom: { windowMinutes }` (the room was engaged with the subject within the
window: a mention of or reply to the subject, or the owner posting), `keywords` (case-insensitive substrings), `senders`
(user ids), `dm` (the message is in a DM of the subject), `notification` (for `notification.created`). Only types are
validated: window and event counts must be positive (`maxEvents` an integer), `cooldownSeconds` 0 or more, and no value has
an upper bound. An unknown key is rejected because a misspelled predicate would otherwise leave `match` empty, which
matches everything.

Always dropped: messages by any bot (the agent's own posts included), edits, system messages, messages the agent sent in its
owner's name, the owner's own messages (they only keep a room warm), and events in the agent room itself. For
`subject: principal` the owner and the bot must both still be members of a mirrored room. Notifications go to a filtered
trigger only when addressed to the subject; the room scope does not apply, and a notification caused by the agent itself is
dropped.

**Owner's DMs and notifications are private.** They are delivered only while the owner is the only person in the agent room.
If a second person joins it, those events are withheld (a rate-limited warning is logged).

**Coalescing and cooldown.** Without `coalesce`, an event inside the cooldown is dropped. With it, a burst becomes one turn
that lists up to `maxEvents` events ("N of M events shown"); a flush inside the cooldown waits for it to end, and a flush
that meets a running turn of the same trigger is delivered after that turn ends. Buffers, cooldowns and the hot-room
memory are process-local: a hub restart loses what was buffered.

Two triggers that cover the owner's day (`promptTemplate` is required in both):

```json
{
  "botId": "<agent bot id>", "type": "event", "promptTemplate": "Triage what needs the owner's attention.",
  "filter": {
    "events": ["message.new"], "subject": "principal", "rooms": "mirrored", "includeDms": true,
    "match": { "mention": true, "reply": true, "hotRoom": { "windowMinutes": 15 }, "dm": true },
    "coalesce": { "windowSeconds": 30, "maxEvents": 10 }
  }
}
```

T1 fires one turn for: the owner mentioned, a reply to the owner, a message in a room engaged within 15 minutes, or a DM to
the owner. It stays silent for a declined or non-mirrored room, a message by a bot, the agent's own post, and an edit.

```json
{
  "botId": "<agent bot id>", "type": "event", "promptTemplate": "An item was assigned to the owner. Summarise it.",
  "filter": { "events": ["notification.created"], "subject": "principal" }
}
```

T2 fires one turn when an item is assigned to the owner by someone else and nothing when the agent assigned it. It has no
`coalesce`, so a second notification inside the 30 s cooldown is dropped; add `coalesce` or `cooldownSeconds: 0` if every
assignment must wake the agent.

Injected turns run in the agent room as a fresh task, with the matched events appended after the prompt as JSON. While a
super agent is live, its platform rules swap the single-room lines for a block that treats chat, list item and notification
content as data and requires owner confirmation for bulk or destructive changes.

Source: `apps/meteor/server/services/agent-trigger-filter.ts` (pure), `agent-event-trigger-handler.ts` (dispatch),
`agent-trigger-injector.ts` (turn start).

## Trigger authorship

Who may create, change, run and remove the triggers of a super agent (`resolveTriggerAccess` in
`agent-trigger-endpoints.ts`):

- the **owner** and **administrators** (`view-user-administration`);
- the **agent itself**, only from a session in its own agent room (the proxy stamps `x-privos-session-room`);
- nobody else: other agent-room owners and leaders, the agent from any other session, and any session bound to another room
  are refused with the usual "not authorized" answer.

A trigger about the owner (`subject: principal`) must be created by the owner or by the agent. An agent-authored trigger
runs with the owner as initiator. Every change records `updatedBy` and `updatedAt` and writes a `super-agent.trigger-change`
row before the change (action, trigger id, type, field names, and `by`: `owner`, `admin` or `agent`; never prompt text).
The agent editing its own triggers is a risk the owner accepted: one injected turn can change a filter or prompt with
lasting effect, and the audit row is how it is noticed.

## CLI bot mode and egress transport

Bot keys cannot use the public `lists.*` and `items.*` routes. In bot mode the `privos` CLI calls the room routes
`/api/v1/internal/rooms/:rid/...` and requires `--room` (default `PRIVOS_ROOM_ID`). Command reference:
[CLI](https://github.com/PrivOS-AI/privos/blob/main/docs/cli/README.md).

Inside an agent VM there is no bot key. With `PRIVOS_SANDBOX_MODE=true`, `PROXY_URL` and `PROXY_TOKEN` set and no credential
of its own, the CLI sends `hub rooms|lists|items|dm` and `agents a2a` requests as `POST $PROXY_URL/egress` envelopes
(`url`, `method`, `headers`, string `body`), authenticated to the proxy with `x-proxy-token` only. The proxy matches the URL
against the agent's catalog and attaches the bot key; it stamps the session room. `hub get`, `subscribe` and
`sandbox tasks answer` never use it.

**Catalog.** The hub's bot-key push writes the platform rows into the proxy catalog. For a flagged bot, and for its
`agent-room-<botId>` project only, it also writes **exact** (never prefix) routes: reads `rooms.get`, `rooms.info`,
`channels.info`, `groups.info`, `channels.members`, `groups.members`, and the room-management writes `channels|groups`
`.create`, `.invite`, `.kick`, `.archive`, `.rename`. Every push reconciles the project's platform rows, so unflagging (or
engaging the kill switch) removes the extra rows on the next push, which the hub triggers itself. Credential-vault rows are
never touched. Lists: `SUPER_AGENT_EGRESS_EXACT_ROUTES` in the sandbox `src/lib/bot-key-egress-catalog.ts`, mirrored by
`SUPER_AGENT_ROOM_ROUTES` in the hub's `bot-bearer-route-policy.ts`; a test pins the two together.

Each room-management write by a flagged bot leaves a `super-agent.room-write` row before it runs; if the row cannot be
written, or the kill switch is engaged, the hub refuses the call (401).

## DM reply as the owner

When a DM wakes the agent, it may ask the hub to answer inside the owner's own one-to-one DM, **as the owner**, with an
agent badge ("sent by <agent>"). The bot never joins the DM.

- The agent calls `agents.superAgent.dmReply` (CLI: `privos hub dm reply --room <dm> --text ...`). The hub accepts it only
  from the agent room session, for a one-to-one DM of the owner with an active person, text only (attachments are
  rejected).
- **Approve (default).** The hub stores a draft and posts a card in the agent room with **Send** and **Discard**. Only the
  owner's click counts; another agent-room member or the bot is refused. Send posts the message through the normal send
  path (size, access and read-receipt rules apply). A newer draft for the same DM replaces the pending one. Drafts expire
  after `SuperAgent_DM_Draft_Expiry_Hours` (default 24); an expired draft cannot be sent.
- **Auto.** The owner switches on "Allow replying to DMs on my behalf" in the agent room's settings panel
  (`bots.superAgent.dmReply.set`). Replies are then posted at once, for every one-to-one DM the owner has, and the agent
  room gets a one-line note with a link. The agent cannot choose or change the mode.
- The marker `sentByAgent` is set only by the hub (any client-supplied value is stripped), keeps the reply from waking the
  bot again, and is what the badge renders. The mobile app and push notifications show no badge.
- Draft text is kept in its own collection, not in the state document that the owner and admins can read.

Every send, discard, supersede and mode switch writes a `super-agent.dm-reply` row (ids, outcome, mode; never the text).
Source: `server/services/super-agent/dm-reply-drafts.ts` and `app/api/server/v1/agents-super-agent-dm-reply.ts`.

## Kill switch

The workspace setting `SuperAgent_Enabled` (default on) is a **freeze**, not a revoke. While it is off:

- item writes, room-management calls from the VM, DM replies (including Send on an existing draft) and the enforce-mode
  allowance are refused;
- triggers with `subject: principal` or `rooms: mirrored` stay silent;
- the egress catalog is re-pushed narrow;
- the mirror pauses (no joins, no leaves); the bot keeps its rooms and roles;
- owner-only turns stay enforced, and clearing a flag is never blocked (a cleared flag is still cleaned up).

Releasing the switch re-pushes the catalog and catches every bot up. Revoking the bot's token is the emergency stop, because
it stops the key itself.

## Audit event types

Durable rows in the server events collection (ids and counts only, never message or prompt text):

| `t` | Written when | Actor |
|---|---|---|
| `super-agent.flag-set` | an admin sets the flag (before the flag is written) | the admin |
| `super-agent.flag-cleared` | an admin clears it | the admin |
| `super-agent.item-write` | an app-owned list item write passes on a super agent's authority | the bot |
| `super-agent.room-write` | a flagged bot's room-management write | the bot |
| `super-agent.trigger-change` | a trigger of a super agent is added, updated (a prompt restore counts) or removed | the caller |
| `super-agent.dm-reply` | a DM reply is sent, discarded or superseded, or the mode is switched | owner, bot or system |

Declared in `packages/core-typings/src/ServerAudit/IAuditSuperAgentEvent.ts` (hub). Query recipe: the ops runbook.

## Where things live

| Concern | Hub path (under `apps/meteor`) |
|---|---|
| Flag routes, eligibility | `app/api/server/v1/bots-super-agent.ts` |
| State, kill switch watcher | `server/services/super-agent/super-agent-state.ts` |
| Settings | `server/settings/bots.ts` (section Super Agent) |
| Mirror, cron | `server/services/super-agent/super-agent-mirror.ts`, `server/cron/superAgentReconcile.ts` |
| Catalog re-push | `server/services/super-agent/super-agent-catalog.ts` |
| Filters and dispatch | `server/services/agent-trigger-filter.ts`, `agent-event-trigger-handler.ts` |
| Trigger routes and authorship | `app/api/server/v1/agent-trigger-endpoints.ts` |
| DM reply | `server/services/super-agent/dm-reply-drafts.ts` |

## Related

- [Agent Rooms](./agent-rooms.md), [Trigger Registry](./trigger-registry.md), [Trigger API Reference](./trigger-api-reference.md)
- [Bot Key & Agent Switching](./bot-key-and-agent-switching.md) (how the key reaches the proxy catalog)
- [Credential Vault](./credential-vault.md) (why a hub PAT is refused)
- [Room-scoped items](../room-scoped-apis/items.md)
