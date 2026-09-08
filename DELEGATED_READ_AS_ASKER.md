# Delegated read as the asker (sandbox agent in the AI Chat window)

When a member talks to a room's **default agent bot** in the **AI Chat window**, the sandbox agent's
hub reads of that room's **isolated lists** run under the **asker's** access instead of the bot's:
owner/admin see everything, a creator / assignee / custom-permission holder sees their own items, a
plain member sees none. Reads only. Everything else the agent does still runs as the bot.

This complements [`PRIVATE_AI_ON_BEHALF.md`](PRIVATE_AI_ON_BEHALF.md) (first-party universal
assistant, DM surface, always on) — this page is the **sandbox agent** (room bot with skills, VM
container) in the **room's AI Chat window**, and it is **off by default**.

## When it applies (all must hold)

| Condition | Why |
|---|---|
| Workspace setting `Agent_Delegated_Read_Enabled` = ON (default OFF) | kill switch; OFF = today's bot view, no redeploy |
| Turn sent from the **AI Chat window** of the room (`aiChatSessionId`, no thread, not a mention) | window sessions are private to their creator; a room-visible reply must not carry one member's slice |
| Room sandbox in **dedicated mode** (not collocated) | a collocated container is shared by several rooms/bots; the grant would leak across them |
| Sender is a current member of the room, human-triggered | membership is re-checked on every read |
| Bot is the room's default agent bot (not an MCP-installed app-agent bot) | app agents keep the app's own authorization model |

Not delegated (agent answers with its own bot access): `@mention` / thread replies in the room,
collocated rooms, on-behalf DM sessions (already read as the human), successor attempts spawned by the
worker (auto-retry, butler resume), `assistant.*` endpoints (header verified then dropped).

## How it works (what you can observe)

1. Hub mints a per-attempt RS256 grant (`typ: privos-delegated-read`, claims `sub` asker, `rid` room,
   `bot`, `att` attempt id, ≤ 15 min) and registers it with the tenant sandbox proxy **before**
   dispatch. The token never enters the container or the attempt payload.
2. The proxy adds `x-privos-delegated-user: <token>` to the agent's hub egress **only** on `GET`
   requests to `/api/v1/internal/rooms/*` and `/api/v1/assistant.*`, and only while at most one
   attempt is live for that project. Anything the container itself sends in that header is stripped.
3. Hub middleware on every `internal/rooms` route verifies the header. Outcomes:

| Situation | Result |
|---|---|
| valid grant, asker still a member, flag ON | read runs with `delegatedViewerUserId = asker` inside the isolated-list filters (nothing else changes) |
| bad signature / wrong `typ` / bearer is not the grant's bot | `403 {"error":"delegated-read-forgery"}` |
| non-GET request carrying the header | `403 {"error":"delegated-read-write-denied"}` |
| grant unknown / revoked / expired, room mismatch, member removed, flag OFF, live attempts > 1 | header dropped, request continues as the **bot**, activity `delegated-read-degraded { reason }` |

4. The grant is revoked when the turn completes/fails/cancels, is parked on AskUser, times out, or the
   flag is switched off. An AskUser answer by the asker re-mints for the same attempt; another member's
   answer runs without a grant.

Turns of one project are serialized while the flag is ON (Redis lock, 120 s wait → `project-busy`).

## What each persona gets

Same table as the on-behalf assistant: owner/admin all items; creator, assignee, `additionalReaders`
/ `additionalEditors` holder their granted items (read cascade to sub-items honored); plain member
none. Non-isolated lists are unaffected.

Known residual: the stored `itemCount` on the list document is still returned, so an agent can
report "the list has N items overall" while showing fewer. Do not rely on it for authorization.

## Client-side notices (AI Chat window activity)

- `delegated-read-degraded { reason }` — reasons `registration-failed | unknown-grant | revoked |
  expired | room-mismatch | membership-lost | kill-switch | collocated | live-attempts`. Copy:
  "Answered with the assistant's own access: <reason>."
- `dedicated-mode-required` — the room is collocated and the bot has skills that need `Bash`;
  switch the room's sandbox to dedicated mode (see
  [`agent-system/agent-rooms.md`](agent-system/agent-rooms.md#sandbox-mode-collocated-vs-dedicated)).

## Window sessions are private

An AI Chat window session belongs to its creator. Listing/reading/sending/streaming on someone
else's window session is refused, and the AI-chat "join" / access-request / auto-approve mechanism
does **not** apply to window sessions (it is self-service and would defeat the boundary).

## What it never does

- Never writes as the asker: item create/update/delete, grants, reassignments stay human actions.
- Never delegates on a room-visible surface (mention, thread).
- Never gives the asker more than their own Lists UI shows.
- Skills are instructed not to persist isolated results into the per-room sandbox workspace; a later
  member's turn in the same dedicated container can read that workspace (accepted residual).

## Rules for skill authors

- Do not cache or write isolated-list results to files; answer from the live read.
- Treat `itemCount` as informational.
- If your skill needs `Bash`/`Task`, document that the room must run in dedicated mode.

## See also

- [`ROOM_CUSTOM_PERMISSIONS.md`](ROOM_CUSTOM_PERMISSIONS.md) — the grants this honors.
- [`internal-apis/lists.md`](internal-apis/lists.md#isolated-lists) — the routes the header applies to.
- Hub settings: `Agent_Delegated_Read_Enabled`; audit codes `delegated-read-ok | degraded | forgery |
  write-denied | minted | revoked`.
