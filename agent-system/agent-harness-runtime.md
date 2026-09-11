# Agent System — Harness Runtime

A second execution runtime for agent bots, alongside PrivOS Sandbox: instead
of the hub driving a sandboxed Claude Code instance it controls, a small
outbound-only bridge CLI on a machine the operator owns
(`@privos_ai/agent-harness`, `privos-app-packages/agent-harness`) drives a
coding-agent harness the operator already runs — Claude Code, Codex, Cursor,
Goose, or a custom [ACP](https://agentclientprotocol.com) command — and
streams the reply back over the same WSS relay it used to receive the turn.

Operator-facing setup, adapter table, permission policies, and isolation
guarantees: `privos-hub/docs/agent-platform/pair-existing-agent-harness.md`.
Mid-turn steering (`turn.steer`), which extends this protocol, is covered in
[Agent Settings UI — Messages during a reply](./agent-settings-ui.md#messages-during-a-reply-mid-turn-steering-policy);
this page only documents `turn.steer`/`turn.steer-result` as protocol entries
and does not repeat the policy semantics.

## Architecture

```
┌──────────────────────────────┐   WSS, Authorization: Bearer <bot token>
│ Operator's machine            │◄────────────────────────────────────────┐
│  privos-agent-harness start   │                                          │
│   ├─ per-room turn queue      │      turn.start / turn.cancel / turn.steer
│   ├─ ACP client (@agentclientprotocol/sdk)  turn.chunk / tool_use / activity / done / steer-result
│   └─ adapter pool: one process per room ──► claude-agent-acp / codex-acp / …
│      (optionally under sandbox-exec/bwrap/docker — see isolation)         │
│  env per room: PRIVOS_URL / BOT_KEY / BOT_ID / ROOM_ID ──► bot REST API ─┼──►┐
└────────────────────────────────────────────────────────────────────────┘   │
                                                                                ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Hub (tenant / self-hosted)                                                        │
│  GET/upgrade /api/v1/agents.harness.relay  ──►  agent-harness-connection-manager  │
│  agents.harness.pairing.{guide,keys,skills}   (public, token-gated, 5-min TTL)    │
│  agents.harness.{status,update,rotatePairing,resetSessions,setRuntime,skills}     │
│  processAIRequestJob (ai-messages.ts)  ──►  agentHarnessChatStreamingCall ──►     │
│    same IPrivOSSandboxChatCallResult contract as privosSandboxChatStreamingCall   │
│  agent-trigger-injector.ts  ──►  agentHarnessChatStreamingCall (own task ids)     │
│  agent-turn-coordinator.ts  ──►  admitTurn()  (steer/queue/interrupt gate)        │
│  GET /api/v1/bot/*  ◄── bot REST API, called by the room process with its env key │
└────────────────────────────────────────────────────────────────────────────────────┘
```

Runtime is selected per agent bot: `customFields.agentRuntime.kind` is
`'sandbox'` (default, absent = sandbox) or `'harness'`
(`apps/meteor/server/services/agent-runtime.ts` `resolveAgentRuntime`). Every
later component — the dispatch switch, the trigger injector, the settings
panel, export/import — reads the runtime through that one resolver rather
than `customFields.agentRuntime` directly.

## Hub ↔ bridge JSON-RPC protocol

Frozen contract, one process on each side:
`apps/meteor/server/services/agent-harness/agent-harness-protocol.ts` (hub's
copy; the bridge package re-declares the same shapes independently — no
cross-repo import). Every frame from the bridge is untrusted network input
and is parsed by a hand-written guard that returns `null` on any malformed
field rather than throwing.

| Method | Direction | Kind | Payload |
|---|---|---|---|
| `harness.hello` | bridge → hub | RPC, first frame | params `{ adapter, adapterVersion?, bridgeVersion, hostname, cwd, permissions, isolation, skillsManifest？: { sandboxVersion }, capabilities: { loadSession }, steering: boolean }` → result `{ agentRoomId, connectUrl?, respondTo }` |
| `turn.start` | hub → bridge | RPC | params `{ turnId, sessionKey, roomId, threadId?, prompt, promptFull, displayPrompt, sender: { _id, username?, name? }, resume, deadlineMs }` → result `{ accepted: true, queued: n }` or error `data.code = 'harness_busy'` |
| `turn.chunk` | bridge → hub | notification | `{ turnId, chunk (accumulated, not a delta), index, isComplete: false }` |
| `turn.tool_use` | bridge → hub | notification | `{ turnId, toolId, toolName, input, status }` |
| `turn.activity` | bridge → hub | notification | `{ turnId, kind, status, message, timestamp }` |
| `turn.done` | bridge → hub | notification, terminal | `{ turnId, status: 'completed' \| 'failed' \| 'cancelled', text, errorMessage?, sessionFresh }` |
| `turn.cancel` | hub → bridge | notification | `{ turnId }` — user Stop or hub timeout |
| `turn.steer` | hub → bridge | RPC | params `{ turnId, roomId, steerId, text, deadlineMs }` → synchronous ack result `{ outcome: 'accepted' \| 'notRunning' \| 'unsupported' }` (never `injected`/`dropped` — those arrive later) |
| `turn.steer-result` | bridge → hub | notification, on the same `turnId` scope | `{ turnId, steerId, outcome: 'injected' \| 'dropped' }` — only `injected` counts as delivered |
| `harness.resetSessions` | hub → bridge | notification | `{}` — bridge forgets stored ACP session ids |

`turn.chunk`'s `chunk` field is the **accumulated** text every time, not a
delta — mirrors the sandbox streaming contract so the same accumulator code
handles both runtimes (`ponytail:` noted ceiling in the bridge source: O(n²)
bytes per turn, switch to deltas if turns exceed ~100 KB).

### `turn.steer` / `turn.steer-result` (plan `260911-0539`)

Added by the mid-turn-steering plan on top of the frozen Phase 1–8 contract.
`turn.steer`'s synchronous RPC result is only ever `accepted` (the bridge
wrote a native steer request to the adapter and is waiting on it),
`notRunning` (the bridge has no turn with that id in flight — the hub retries
once against a rotated attempt id, else repairs the doc and runs a plain
turn), or `unsupported` (the connected adapter for this process never
advertised `_meta.steering.supported` at `initialize` — the bridge **never
probes** an unknown ACP method, because `codex-acp` answers one with a bare
`{}` JSON-RPC success that would be misread as delivered). The eventual
`injected`/`dropped` outcome always arrives as the separate `turn.steer-result`
notification, routed by the connection manager to a **one-shot sink** keyed
`${botId}:${turnId}:steer:${steerId}` — a map entirely separate from the
turn's own primary `TurnSink`, so a steer request can never displace or steal
the sink that owns the turn's chunks/done event. On `injected`, the
connection manager also calls the turn's primary sink's `extendDeadline()`
with the `deadlineMs` captured when the steer was registered (the wire
notification itself never repeats `deadlineMs`).

`harness.hello.steering` is the adapter's **declared** capability
(`claude`/`codex` → `acp-extension`, everything else → `none`), informational
only for the hub's connect message and `doctor` — the real, per-process gate
is what `AcpSession` captures from the adapter's own `initialize` response at
runtime.

## Dispatch switch points

Two call sites choose between the sandbox and harness runtimes, both reading
`resolveAgentRuntime`, both producing the **same** result contract so
everything downstream (room mirror, thread state, cancel, triggers) is
runtime-agnostic:

- **`processAIRequestJob`** (`apps/meteor/app/api/server/v1/ai-messages.ts`)
  — the BullMQ `ai-requests` worker for every mention / AI-chat-window
  message. Calls `agentHarnessChatStreamingCall` for a harness bot,
  `privosSandboxChatStreamingCall` otherwise. This is also where the
  mid-turn steering coordinator sits: `agent-turn-coordinator.ts`'s pure
  `admitTurn()` gate decides `run` / `run-merged` (cancel+merge fallback) /
  `steer` (native) / `wait-then-run` (queue) before the dispatch call, using
  a DB-backed in-flight lookup (`AIMessages`, not per-process state) so it
  works the same whether the second message lands on the hub instance
  running the first turn or a different one.
- **`injectTriggerMessage`** (`apps/meteor/server/services/agent-trigger-injector.ts`)
  — cron/webhook/event triggers. Checks `resolveAgentRuntime(botUser.customFields).kind === 'harness'`
  and calls `agentHarnessChatStreamingCall` directly with a trigger-scoped
  turn id (`trigger-${triggerId}-${timestamp}${random}`) and its own
  autonomous-run system context, entirely outside the `AIMessages` in-flight
  tracking the steering coordinator reads — trigger runs never compete with
  or get steered by a chat message and vice versa. A trigger has no live
  sender to check `respondTo` against, so a harness trigger whose creator is
  not the bot's owner is skipped at fire time unconditionally — it never
  widens past the owner the way `agent-room-members`/`everyone` would for a
  chat message.

For a `respondTo` gate that fails (no permission, or the bridge is offline)
both call sites write a message naming the fix (`"On the paired machine run:
privos-agent-harness start"`, or the owner-only notice) rather than leaving
the message hanging in a sending state.

## Relay lifecycle: connect, presence, disconnect

`GET /api/v1/agents.harness.relay` (`agent-harness-relay-endpoint.ts`) is a
raw WebSocket upgrade, authenticated **only** by `Authorization: Bearer
<bot token>` — deliberately no `?token=` query fallback, so a bot token never
reaches an access log. Upgrade-time checks, each a distinct terminal
response the bridge's reconnect logic treats as non-retryable: missing/
invalid/inactive token → `401`; not an agent bot, or an agent bot whose
runtime isn't `'harness'` (including right after a runtime switch back to
sandbox) → `403`. A connection-attempt rate limit (5/min per token hash) sits
in front of `BotTokenService.validateToken`.

After upgrade, the first frame must be `harness.hello` within 10 s (else
close `4408`). `agentHarnessConnectionManager.register()` replaces (closing
`4409`, `"replaced by <hostname>"`) any existing connection for that `botId`
— **one live bridge per bot, ever** — and the hub posts a system message into
the agent room for both the old socket's replacement and the new one's
connect, naming adapter/hostname/cwd/permissions/isolation, so the owner sees
every takeover. `harness.hello`'s RPC result carries `{ agentRoomId,
connectUrl?, respondTo }`; `connectUrl` is the same `PRIVOS_CONNECT_URL` the
sandbox bot-key push already reads, present only when the deployment has one
configured.

**Presence** (`customFields.agentRuntime.harness.{bridge,lastSeenAt,pairedAt}`)
is written at exactly two points — hello and close — never on a timer or
ping, throttled to one `Users` write per bot per 60 s even across a
crash-loop of hello/close pairs. `isOnline`/`agents.harness.status`'s
`online` field always comes from the **in-memory** connection map, not from
`lastSeenAt`, which is purely informational during the throttle window.

**Disconnect fails every in-flight turn immediately** rather than waiting out
the hub's own turn timeout: `teardownEntry` (the single funnel for replace /
explicit close / natural close) calls `failTurnSinksForBot`, which rejects
every `${botId}:*` primary `TurnSink` with `onDisconnected('harness
disconnected')`, resolves every `${botId}:*` `turn.steer-result` sink as
`dropped`, and resolves every pending `waitForTurnDone` watcher — so a socket
drop mid-turn (crash, network loss, `4401`/`4409`) surfaces to the caller
right away instead of a silent hang until deadline.

`closeByBotId(botId, 4401, reason)` is the forced-close path used by *Rotate
harness pairing* and the harness → sandbox runtime switch; `4409` is reserved
for the "replaced by another bridge" case above.

## Pairing, keys, and skills endpoints

All under `app/api/server/v1/agent-harness-pairing-endpoints.ts` and
`agent-harness-endpoints.ts`, backed by `agent-harness-pairing-store.ts`
(in-memory, per-bot, 5-minute TTL) and `agent-harness-pairing-guide.ts`
(single source of the guide text — the public page, the JSON the bridge
reads, and the short message posted into the agent room all render from the
same functions so the three surfaces never drift):

| Route | Auth | Purpose |
|---|---|---|
| `GET /agent-harness/pair/:token` | none (public page, client route) | Renders `agents.harness.pairing.guide`'s JSON; never shows the keys/skills URLs |
| `GET /v1/agents.harness.pairing.guide?token=` | none | Steps, isolation advice, `respondTo`, `keysUrl`, `skillsUrl` — no `botId`/`roomId`/`hubUrl` (those only ever travel through the one-time `keys` link) |
| `GET /v1/agents.harness.pairing.keys?token=` | none, single-use | Returns `{ hubUrl, agentId, botId, agentRoomId, botToken }` exactly once (atomic consume, no `await` between the non-consuming peek and the consume — the ordering that makes "two concurrent calls, one 200" hold); a second call fails `pairing_keys_already_retrieved` |
| `GET /v1/agents.harness.pairing.skills?token=` | none, multi-use until expiry | Proxies the cached skills `.tgz` |
| `GET /v1/agents.harness.status` | Meteor session, owner or `view-user-administration` | Runtime, adapter, `online`, `respondTo`, `lastSeenAt`, `bridge` info |
| `POST /v1/agents.harness.update` | same | Sets `respondTo`; requires `force: true` to widen past `owner` while the connected bridge reports `isolation: 'prompt' \| 'none'` |
| `POST /v1/agents.harness.rotatePairing` | same | Invalidates the bot token, force-closes the live socket (`4401`), mints a fresh 5-minute guideline |
| `POST /v1/agents.harness.resetSessions` | same | Notifies `harness.resetSessions`; `notified: false` if offline |
| `POST /v1/agents.harness.setRuntime` | same, plus `isEligibleForHarnessRuntime` for `sandbox → harness` | Exclusive per-agent runtime switch; the bot token is **kept** across the switch either direction |
| `GET /v1/agents.harness.skills` | Bearer bot token of a harness agent (not a Meteor session) | The bridge's `skills update` source |

Every pairing route is `Cache-Control: no-store`, rate-limited 10/min **per
token and per client IP** (either bucket tripping blocks the request), and
never logs the token.

`agents.harness.pairing.guide/keys/skills` and `agents.harness.status` are
the only routes any of this reaches without a Meteor session or a bot token;
every mutating route (`update`, `rotatePairing`, `resetSessions`,
`setRuntime`) shares one gate, `assertHarnessAgentManager`
(`agent-harness-authorization.ts`): the bot's `_createdBy` **and**
`edit-bot`, or an admin with `view-user-administration` — never bare
`edit-bot`, which the ordinary `user` role also holds.

## Skills bundle source (SSRF-safe)

`agent-harness-skills-bundle.ts` `fetchAgentHarnessSkillsBundle()` is the
single fetcher behind both the pairing guideline's skills link and
`agents.harness.skills`. It reads the sandbox URL/API key from the
**workspace-global** sandbox setting only
(`getOptionalGlobalPrivOSSandboxSettings`) — deliberately **not**
`getEffectivePrivOSSandboxConfig`, which would also honour a room owner's
sandbox override URL. Because this bundle is reachable from a 5-minute
pairing link a non-owner could be holding, or from any harness bot's own
token, honouring a per-room override would let that link holder make the hub
fetch an attacker-chosen URL server-side. The fetch requires
`Content-Type: application/gzip`, streams the body capped at 2 MB (so a
lying or absent `Content-Length` can't bypass the cap), and caches the result
in memory for 10 minutes — cache isolation across tenants comes from
topology (one hub process per tenant), not from a cache key. `No workspace
sandbox configured` surfaces as `503 skills_bundle_unavailable` on both
routes.

The sandbox side is one read-only, key-gated route,
`GET /api/skills/template-bundle` in `privos-sandbox-mt` (`x-api-key` gated
like every sandbox `/api/*` route) — the only change this feature makes to
the sandbox repo. It streams a `.tgz` of the same
`src/hooks/template/skills/*` + `template/agent-room/skills/*` +
`packages/skill-sdk/lib/privos_skill.py` a sandbox project's own bot gets,
minus `privos-landing-page` (excluded — needs sandbox-provided
`PRIVOS_CONTACT_CONFIG` the harness workspace has no source for) via an
`exclude` query param, plus a `MANIFEST.json` the bridge's `skills update`
compares against to skip a no-op re-download.

## Per-room isolation model

A harness agent's workspace root is `~/privos-harness/<agentId>/` on the
operator's machine — outside the bridge's own hidden config directory
(`~/.privos/agent-harness/`), which every isolation level hides from a room
process. Layout: `.privos/skills` + `.privos/skill-sdk` (shared, read-only
from rooms), one agent-global `IDENTITY.md` (read-only from rooms — the
`agent-bot-edit identity` skill resolves it via its own **realpath**, since
`.privos/skills` is a symlink), and `rooms/<roomId>/` — one directory per
room the agent is mentioned in, each served by its **own adapter subprocess**
(pooled, cap `--max-rooms`, default 8, LRU-idle-reaped after 10 min; a reap
without `loadSession` support makes the room's next turn `sessionFresh`,
covered by `promptFull`).

Isolation is a spectrum the bridge self-tests into (`--isolation auto`,
default): it spawns a real adapter process under each candidate wrapper in a
scratch room and asserts, from inside, that it can write in the room, cannot
read a sibling room or the real credential directories, and can still reach
the shared skills bundle — never trusting binary presence alone.

- **`wrap`** — per-room OS sandbox the bridge generates itself: macOS
  `sandbox-exec` (Seatbelt, `src/isolation/seatbelt-profile.ts`) or
  Linux/WSL2 `bwrap` (`src/isolation/bwrap-args.ts`). Both deny-read the
  real `~/.claude`, `~/.claude.json`, `~/.codex`, `~/.ssh`, `~/.aws`,
  `~/.gnupg`, and the bridge's own `~/.privos`, then re-allow only the one
  room directory, the shared `.privos`, and `IDENTITY.md`. Every path is
  `realpath`'d before being emitted into the profile (Seatbelt matches
  canonical paths only), and an ancestor between a deny root and an allowed
  child gets `file-read-metadata` so `chdir`/spawn-with-`cwd` still works.
  Nested sandboxes are EPERM, so the adapter's own native sandbox (Claude
  Code's `sandbox.enabled`, Codex's `workspace-write`) is turned **off**
  under `wrap`.
- **`container`** — `docker run --rm -i --name privos-room-<roomId>` per
  room; the same file boundary via bind mounts instead of a kernel sandbox.
- **`prompt`** — no OS sandbox passed the self-test. Falls back to the
  adapter's own native sandbox (now the only real enforcement) plus an
  `<isolation_policy>` section in the standing preamble and the room's
  `CLAUDE.md`/`AGENTS.md` instructing the model to stay inside its room.
  Widening `respondTo` past `owner` while a live bridge reports `prompt` (or
  `none`) requires the caller to resend `agents.harness.update` with
  `force: true` — the hub's own `harness_isolation_prompt` coded error is
  what the settings panel turns into that confirmation flow.
- **`none`** — explicit-only, no wrapper, no policy section.

Independent of the level above, each room's adapter process gets its own
**adapter state directory** (`rooms/<roomId>/.home/`), seeded only with the
one credential file that adapter declares in its `credentialFiles[]`
(`acp/adapter-table.ts`) — Claude's `.credentials.json` via
`CLAUDE_CONFIG_DIR`, Codex's `auth.json` via `CODEX_HOME` — never its
settings, hooks, or MCP config, so a hook planted by one room's process never
loads in another room's. Adapters with no documented credential file (Cursor,
Goose, `custom`) keep their real, shared state dir; the level reported back
to the hub (in `harness.hello.isolation`, `agents.harness.status`, and the
runtime panel) downgrades to `wrap-shared-state` for those — file isolation
still holds, only that adapter's own login/config is shared, same as an
unsandboxed run. The credential the PrivOS skill SDK itself uses
(`PRIVOS_BOT_KEY`) stays **agent-global and static in env** at every
isolation level (a deliberate trade-off recorded in
`plans/260910-1648-pair-agent-with-existing-harness-via-acp-bridge/plan.md`
validation sessions 6–8): cross-room *file* access is bounded by the OS,
cross-room *API* access is bounded by the hub's `respondTo` gate and
membership rules only, same as the reference platform this design is modeled
on.
