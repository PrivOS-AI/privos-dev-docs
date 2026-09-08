# Private "On my behalf" AI Chat

A room member can open a **private AI-chat session about the current shared room**, in which the
first-party universal assistant answers using **that member's own access** to the room — their
isolated-list items and their custom-permission grants (`additionalReaders`/`additionalEditors`) —
and nobody else can see the conversation. Reads only: the assistant never writes on the member's
behalf.

## Why

The shared-room assistant answers as the **bot** and is confined to the room's shared surface
(isolated items excluded). That is correct for a shared conversation, but a member often wants to ask
about *their own* slice ("what's assigned to me here?", "summarise the items I can see"). The private
on-behalf session gives each member a personal, permission-accurate assistant for a shared room
without exposing anyone else's private items.

## Identity model (how it stays safe)

- **Anchor = the member's existing universal-bot Assistant DM.** There is no hidden room and no
  special flag on any room. A private session is an `AIChatSession` record with `ownerUserId` (the
  member), `contextRoomId` (the shared room being discussed), and an immutable `onBehalf: true`.
- **Owner-gated end to end.** Every session-addressed route (list/read/send/retry/title/context/stats/
  streaming) requires `ownerUserId === caller`; membership of the shared room is NOT sufficient. A
  session's `ownerUserId`/`contextRoomId`/`onBehalf` cannot be changed after creation, and a legacy
  session cannot be flipped into on-behalf.
- **Reads run as the human viewer.** On the private (DM) surface the assistant reads execute as the
  member via `assertUserRoomAccess(viewer, roomId)` + `filterItemsForIsolatedList(..., viewer)`, so
  the result is exactly the member's own UI slice — isolated visibility and custom-permission grants
  (incl. ancestor cascade) honored automatically. The member can never see more than their own UI.
- **Context is hub-supplied, membership re-checked per turn.** The recent-messages context of
  `contextRoomId` is injected server-side only after re-verifying the member still belongs to that
  room on that turn; any client-supplied context is ignored. Remove the member from the room and the
  next turn gets no context and no reads.
- **Private audience.** Streaming/emit is owner-keyed via the member's DM; no other user's client
  ever receives the session's events.

## What the agent can answer (per persona)

Ask the same question about an isolated list and each member gets exactly their slice:

| Persona | Sees |
|---|---|
| creator | items they created |
| assignee | items assigned to them |
| `additionalReaders` holder | items granting a permission they hold (read) |
| `additionalEditors` holder | items granting a permission they hold (read+edit visibility) |
| room owner/admin | all items |
| plain member | only shared (non-isolated) items |

Two access-explanation read endpoints back "who can see this?" questions:
- `assistant.get-room-permissions { roomId? }` → the room's custom-permission catalog (any member).
- `assistant.get-item-access { itemId }` → the item's base rule + the permission names in
  `additionalReaders`/`additionalEditors`. Member **identities** (holders) are returned only to a
  human room owner/admin, mirroring `rooms.customPermissions.members`.

## What it cannot do — reads only

The on-behalf agent **never writes as the member**: no creating/updating items, no setting
`additionalReaders`/`additionalEditors`, no reassigning, no deleting on the member's behalf. Writes
remain a human action (UI session), because autonomous write-as-a-user requires a per-attempt
credential the transport does not carry. See [`ROOM_CUSTOM_PERMISSIONS.md`](ROOM_CUSTOM_PERMISSIONS.md)
("Limitation — no agent write-on-behalf"). (An agent given the room `owner`/`admin` role can write as
**itself** — a separate, deliberate config, not this feature.)

## Availability

Always on wherever the universal assistant is available — there is no separate feature flag. The
control appears for any member inside a shared (non-DM) room, and every turn is still gated by
`canAccessUniversalAssistant` (the universal-assistant master switch + per-user allowlist). Disabling
the universal assistant disables this along with it.

## Related: the sandbox room bot in the AI Chat window

The same per-persona result is available from a room's **default agent bot** (sandbox agent with
skills) when a member uses the room's **AI Chat window**, via a per-attempt read grant instead of a
DM surface. Unlike this feature it is **off by default** (`Agent_Delegated_Read_Enabled`), requires
the room's sandbox in **dedicated mode**, and never applies to `@mention`/thread replies. See
[`DELEGATED_READ_AS_ASKER.md`](DELEGATED_READ_AS_ASKER.md).
