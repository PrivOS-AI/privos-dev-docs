# Trigger API Reference

REST endpoints for managing agent triggers and receiving webhooks.

All endpoints except `agents.webhook/:token` require authentication via `X-Auth-Token` header.

## Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/v1/agents.triggers.list` | Yes | List triggers for an agent |
| POST | `/v1/agents.triggers.add` | Yes | Add a new trigger |
| POST | `/v1/agents.triggers.update` | Yes | Update an existing trigger |
| POST | `/v1/agents.triggers.remove` | Yes | Remove a trigger |
| POST | `/v1/agents.triggers.run` | Yes | Manually fire a trigger |
| POST | `/v1/agents.webhook/:token` | No | Receive external webhook |
| POST/GET | `/v1/agents.a2a.*` | Bot key (Bearer) | Bot-to-bot messages between roster agents (`send`, `team.members`, `list`); see [Bot-to-Bot Protocol](./bot-to-bot-protocol.md) |

What a trigger does after it fires (a handler script, the quiet `NO_REPORT` turn, the outcome fields) is described in
[Agent Routines](./agent-routines.md); this page is the request and response reference.

## Authorization

All authenticated endpoints verify **bot ownership**: the requesting user must be the bot's `_createdBy` user OR have `admin` permission.

A [super agent](./super-agent.md) adds a stricter rule for `list`, `add`, `update`, `remove`, `run` and the prompt-history
restore: only its owner, an administrator (`view-user-administration`) or the agent itself from a session in its own agent
room may call them. Any other caller, and any session bound to another room, gets `Bot not found or not authorized`. Every
`add`, `update` and `remove` on a super agent also writes a `super-agent.trigger-change` audit row.

---

## GET /v1/agents.triggers.list

List all triggers for an agent. Known gap: the response currently includes each webhook trigger's `webhookSecret`; treat the list output as secret.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `botId` | string | Yes | Agent bot user ID |

**Response:**

```json
{
  "triggers": [
    {
      "id": "abc123",
      "type": "cron",
      "enabled": true,
      "schedule": "every_1h",
      "prompt": "Scan trending news",
      "lastRunAt": "2026-04-16T10:00:00.000Z"
    },
    {
      "id": "def456",
      "type": "webhook",
      "enabled": true,
      "prompt": "Process incoming data",
      "webhookToken": "xyz789",
      "lastRunAt": null
    },
    {
      "id": "ghi012",
      "type": "event",
      "enabled": true,
      "event": "message.new",
      "sourceRoomId": "GENERAL",
      "promptTemplate": "Triage this message",
      "lastRunAt": null
    }
  ],
  "success": true
}
```

---

## POST /v1/agents.triggers.add

Add a new trigger. Maximum 20 triggers per agent.

**Body Parameters (common):**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `botId` | string | Yes | Agent bot user ID |
| `type` | string | Yes | `cron`, `webhook`, or `event` |
| `name` | string | No | Title shown on the settings card, under 100 characters |
| `description` | string | No | One line shown under the title |
| `nextAction` | string | No | What a fire does: `agentic_response` (default), `run_handler`, or `emit_event` (webhook only). See [Running a handler](#running-a-handler-nextaction-and-handler) |
| `handler` | object | With `run_handler` | `{ "name": "<service>" }`, the room service to run first |

**Body Parameters (type=cron):**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `schedule` | string | Yes | A preset key, a 5-field cron expression or a one-time instant `at:<ISO-8601>`; see [Schedules](#schedules) |
| `timezone` | string | No | IANA name for a recurring cron (for example `Asia/Bangkok`); absent means UTC. Dropped when the schedule is one-time |
| `prompt` | string | Yes | What the agent should do (max 500 chars) |

**Body Parameters (type=webhook):**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | Yes, except `emit_event` | What the agent should do with incoming data (max 500 chars) |

`webhookToken` and `webhookSecret` are auto-generated on creation.

**Body Parameters (type=event):**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `event` | string | Yes, unless `filter` is set | Event type (see supported events) |
| `sourceRoomId` | string | No | Filter to specific room (empty = all rooms) |
| `filter` | object | No | Subscription filter, see [Subscription filter](#subscription-filter). Replaces `event` and `sourceRoomId(s)` |
| `promptTemplate` | string | Yes | What the agent should do (max 500 chars) |

**Valid Schedules:** see [Schedules](#schedules).

**Valid Events:**

`message.new`, `message.edited`, `message.deleted`, `message.mention`, `message.bot_mention`, `room.joined`, `room.left`, `user.joined`, `user.left`, `list.item.created`, `list.item.deleted`, `list.item.stage_changed`, `list.item.attributes_changed`, `file.created`, `file.updated`, `file.deleted`, `folder.created`, `folder.deleted`, `folder.renamed`, `notification.created`

`notification.created` (an in-app notification stored for a user) is accepted only inside a `filter`: a plain trigger on it
would receive every user's notifications, so `add` and `update` refuse it and the dispatcher ignores any stored one.
Source of truth: `VALID_EVENTS` in `apps/meteor/app/api/server/v1/agent-trigger-endpoints.ts`.

### Subscription filter

`filter` (event triggers only) makes the hub decide, before it injects a turn, whether an event matters. It is data, not
code; the hub never runs anything the filter contains. Semantics, the owner-only DM rule, coalescing and two worked examples
(T1: messages about the owner; T2: assignments to the owner) are in [Super Agent](./super-agent.md#subscription-filters).
A filtered trigger can also run a handler script that screens each batch before any model turn; see
[Agent Routines](./agent-routines.md).

| Field | Type | Description |
|-------|------|-------------|
| `events` | string[] | Required, non-empty. `message.new` and/or `notification.created`. This replaces the trigger-level `event` |
| `subject` | string | `bot` (default) or `principal` (the agent's owner; super agents only) |
| `rooms` | string[] or `"mirrored"` | Room ids you can access, or the rooms a super agent mirrors. For `principal` the default is `mirrored` and explicit ids must be mirrored rooms |
| `includeDms` | boolean | `principal` only, default `true`: also evaluate the owner's one-to-one DMs |
| `match` | object | Any-of predicates: `mention`, `reply`, `dm`, `notification` (booleans), `hotRoom` (`{ windowMinutes }`), `keywords` (strings), `senders` (user ids). Empty matches every event in scope. Unknown keys are rejected |
| `coalesce` | object | `{ windowSeconds, maxEvents }`: deliver a burst as one turn |
| `cooldownSeconds` | number | Minimum seconds between turns, default 30, `0` allowed |

Rules enforced by `add` and `update`:

- `subject: principal` and `rooms: "mirrored"` need a super agent; a trigger about the owner must be created by the owner
  or by the agent.
- `filter` cannot be combined with `event` (or, on `add`, with `sourceRoomId` / `sourceRoomIds`). A filtered trigger
  stores a rollback marker in `sourceRoomIds` so a hub without filter support never fires it; the current hub ignores it,
  and `update` ignores room-scope fields on a filtered trigger instead of applying them.
- Only types are validated. Counts and windows must be positive, and no value has an upper bound.
- A stored filtered trigger returns `filter` (normalised: defaults filled, keywords lower-cased) instead of `event`.

**Response (cron/event):**

```json
{
  "success": true,
  "trigger": {
    "id": "abc123",
    "type": "cron",
    "enabled": true,
    "schedule": "every_1h",
    "prompt": "Scan trending news"
  }
}
```

**Response (webhook):**

```json
{
  "success": true,
  "trigger": {
    "id": "def456",
    "type": "webhook",
    "enabled": true,
    "prompt": "Process incoming data",
    "webhookToken": "xyz789",
    "webhookSecret": "secret..."
  },
  "webhookUrl": "https://chat.example.com/api/v1/agents.webhook/xyz789"
}
```

**Errors:**

| Error | Cause |
|-------|-------|
| `botId and type are required` | Missing required fields |
| `type must be cron, webhook, or event` | Invalid type |
| `Bot not found or not authorized` | Bot doesn't exist or user doesn't own it |
| `Maximum 20 triggers per agent` | Limit reached |
| `Invalid schedule. Use a preset (every_5m, every_1h, ...), a 5-field cron expression (e.g. 0 */3 * * *) or a one-time instant (at:<ISO-8601>)` | `schedule` is none of the three forms, or is missing |
| `One-time schedule must be in the future (at:<ISO-8601>, e.g. at:2026-10-11T11:00:00.000Z)` | An `at:` instant that is not after now |
| `Invalid timezone. Use an IANA name (e.g. Asia/Bangkok)` | `timezone` is not a name the runtime knows |
| `prompt is required for cron triggers` | Empty prompt |
| `Invalid event` | Event not in valid events list |
| `promptTemplate is required for event triggers` | Empty prompt template |
| `notification.created needs a filter` | `notification.created` used without a `filter` |
| `filter.<field> ...` | A filter field has the wrong type or is not a known key |
| `Listening about the owner or in 'mirrored' rooms needs a super agent` | `subject: principal` or `rooms: "mirrored"` on an ordinary agent |
| `A filtered trigger lists its events and rooms in the filter, not in event or sourceRoomIds` | `filter` sent together with `event`, `sourceRoomId` or `sourceRoomIds` |
| `A trigger about the owner must be created by the owner or by the agent` | `subject: principal` created by someone else |
| `name must be under 100 characters` | `name` is too long |
| `name is required for emit_event webhooks` | `emit_event` webhook without a `name` |
| `prompt is required for <action> webhooks` | A webhook whose action runs the agent has no `prompt` (for example `prompt is required for run_handler webhooks`) |
| `nextAction must be agentic_response, emit_event or run_handler` | Unknown `nextAction` |
| `nextAction is only valid for webhook triggers` | `emit_event` on a cron or event trigger |
| `handler.name must match ^[a-z0-9][a-z0-9-]{0,39}$` | A `handler` was sent with a bad name, whatever `nextAction` is |
| `run_handler needs a handler: { name }` | `run_handler` without `handler` |
| `handlers run in the PrivOS Sandbox; this agent runs on a harness` | `run_handler` on a harness agent |
| `a room-scoped cron runs threads; a handler needs the global cron` | `run_handler` on a cron with `sourceRoomId(s)` |
| `run_handler webhooks require the secret` | `run_handler` webhook without a stored secret |
| `only the owner, an admin or the agent in its room may attach a handler` | Caller is none of the three (`error-not-authorized`) |
| `handler-not-found: no service "<name>" is declared in the agent room` | No such service in the agent room's project (`errorType` `handler-not-found`) |
| `handler-not-found: service "<name>" is not a handler (kind <kind>)` | The service exists with another kind (`errorType` `handler-not-found`) |
| `handler-check-unavailable: the agent's runtime could not be asked for its services (<why>); try again` | The runtime could not be asked; `<why>` is `no-sandbox`, `harness`, `no-bot-key`, `collocated`, `http-<status>`, `aborted`, `network` or `invalid-response` (`errorType` `handler-check-unavailable`) |

From an agent, the `agent-scheduler` skill builds this body: `trigger.js add --type event --filter '<json>'` (or
`--filter-file <path>`) with `--prompt` or `--prompt-file`. It refuses `--event` and `--room` next to a filter locally with
the hub's message above.

---

## Schedules

The `schedule` of a cron trigger is one string, in one of three forms. Grammar and messages:
`apps/meteor/server/lib/cron-schedule-utils.ts`.

| Form | Example | Notes |
|---|---|---|
| Preset key | `every_1m`, `every_2m`, `every_3m`, `every_4m`, `every_5m`, `every_15m`, `every_30m`, `every_1h`, `every_6h`, `every_12h`, `every_24h` | Kept as written |
| 5-field cron expression | `0 */3 * * *` (every 3 hours), `*/10 * * * *`, `30 9 * * 1-5` | Any step `cron-parser` accepts; kept as written |
| One-time instant | `at:2026-10-11T18:00:00+07:00` | `at:` plus an ISO-8601 date-time with `Z` or an offset; a date-time without an offset is read as UTC. Stored normalised to UTC (`at:2026-10-11T11:00:00.000Z`) |

**`timezone`** (cron only, optional). An IANA name such as `Asia/Bangkok`. A recurring cron is evaluated in that zone; with no
`timezone` it is evaluated in UTC, so `0 9 * * *` fires at 09:00 UTC (16:00 in Bangkok). A stored trigger never moves: the
zone applies only when it is written. `timezone` is not meaningful for a pure interval (`0 */3 * * *`). A one-time schedule
has no timezone: it is dropped on `add`, removed on `update`, and not validated. `update` with `timezone: null` removes a
stored zone. `agents.triggers.list` returns `timezone` when one is stored.

**One-time semantics.**

- The heartbeat runs every minute. A one-time trigger fires once, on the first tick at or after its instant, so it can fire up
  to a minute late. An instant missed while the hub was down fires late on the next tick.
- The same write that stamps `lastRunAt` also sets `enabled: false`. The trigger stays in the list as history: disabled, with
  `lastRunAt`. Enabling it again does **not** fire it again.
- `update` with a new `at:` value clears `lastRunAt` and sets `enabled: true` (unless the same call sends `enabled: false`),
  which re-arms it. Editing a recurring schedule keeps `lastRunAt`.
- A past instant is refused on `add` and `update` with
  `One-time schedule must be in the future (at:<ISO-8601>, e.g. at:2026-10-11T11:00:00.000Z)`. Agent export and restore keep
  `at:` and `timezone`; a restored past `at:` is skipped with `Trigger "<label>": one-time schedule already passed, skipped`.

The same grammar is offered by the trigger form and the inline editor ([Agent Settings UI](./agent-settings-ui.md)), by the
agent builder chat ([Agent Builder](./agent-builder.md)) and by `trigger.js`
([Self-Management Skills](./self-management-skills.md)). The hub never parses natural language: clients compute the instant.

---

## Running a handler (`nextAction` and `handler`)

Every trigger type can run a room service before, or instead of, the model turn. Behaviour, exit codes and fallback:
[Agent Routines](./agent-routines.md#the-check-stage-handlers).

| Field | Rule |
|---|---|
| `nextAction` | **cron** and **event**: `agentic_response` (the default; not stored) or `run_handler`. **webhook**: `agentic_response` (default, stored), `run_handler` or `emit_event`. `emit_event` stays webhook-only and unchanged |
| `handler` | `{ "name": "<service>" }`; only the name is stored. The name must match `^[a-z0-9][a-z0-9-]{0,39}$` on **every** write that carries a `handler`, whatever `nextAction` is. A flip to `run_handler` needs a valid `handler`, new or already stored |
| prompt field | Still required with `run_handler`: `prompt` (cron, webhook) or `promptTemplate` (event). It is what the model turn runs on a wake or a fallback |

`update` accepts `nextAction` and `handler` for every type. `nextAction: agentic_response` puts the agent back in charge and
leaves the stored `handler` unused. The `emit_event` flip keeps its rule that `name` must exist
(`name is required when switching to emit_event`); the other flips need the type's prompt field
(`<prompt|promptTemplate> is required when switching to <action>`).

**Refused when `run_handler` is set**, in this order (messages in the errors table above): a missing handler, a bad name,
a harness agent, a room-scoped cron (in either direction: adding a room scope to a handler cron is refused too, a scope
cleared in the same write is accepted), a webhook without a secret.

**Who may wire one.** Only the bot's owner, a holder of `view-user-administration`, or the agent from a session in its own
agent room. This check and the lookup of the handler in the agent room's project run only when the write sets `nextAction`
or `handler`, so an edit of a description or the `enabled` toggle on a handler trigger is neither refused nor needs the
runtime awake.

**Webhook receiver.** A `run_handler` webhook is refused with 401 when the trigger has no stored secret or the presented
secret is wrong, and counts against the same 60 per minute limit. The handler receives `[{ headers, body, senderIp }]`.

**Manual run.** `agents.triggers.run` goes the same way a scheduled fire goes, so "Fire now" exercises the handler.

### Fields the hub writes on the trigger

Returned by `agents.triggers.list`; never accepted on `add` or `update`.

| Field | Meaning |
|---|---|
| `lastRunAt` | When the trigger last fired (cron and event: before the dispatch; webhook and manual run: after it) |
| `lastOutcome` | `handled`, `woke`, `fallback`, `paused`, `quiet` or `posted`; see [outcome codes](./agent-routines.md#outcome-codes) |
| `lastError` | The failure code behind a `fallback` or `paused` (`exit-1`, `timed-out`, `owner-disabled`, …). Cleared by a `handled` or `woke` run; never holds output or payload |
| `consecutiveFallbacks` | Fallbacks in a row; at 3 the next failing fire is `paused`. Reset by a successful run or any `update` |
| `updatedBy`, `updatedAt` | The last editor |

```json
{
  "id": "abc123",
  "type": "cron",
  "enabled": true,
  "schedule": "0 9 * * *",
  "timezone": "Asia/Bangkok",
  "prompt": "Follow AgentFiles/routines/morning-digest.md",
  "nextAction": "run_handler",
  "handler": { "name": "screen-feed" },
  "lastRunAt": "2026-10-11T02:00:00.000Z",
  "lastOutcome": "fallback",
  "lastError": "timed-out",
  "consecutiveFallbacks": 1
}
```

---

## POST /v1/agents.triggers.update

Update fields on an existing trigger.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `botId` | string | Yes | Agent bot user ID |
| `triggerId` | string | Yes | Trigger ID |
| `schedule` | string | No | New schedule (cron only): preset, cron expression or `at:` instant; see [Schedules](#schedules) |
| `timezone` | string or null | No | IANA name for a recurring cron; `null` removes it. Ignored for a one-time schedule |
| `prompt` | string | No | New prompt (max 500 chars). Cron and webhook triggers |
| `promptTemplate` | string | No | New prompt template (max 500 chars). Event triggers |
| `enabled` | boolean | No | Enable/disable toggle |
| `event` | string | No | New event type (event only; refused on a filtered trigger) |
| `sourceRoomId`, `sourceRoomIds` | string, string[] | No | New source room scope |
| `filter` | object | No | Replace the subscription filter (event only), same shape as on `add` |
| `name`, `description` | string | No | Card title and subtitle |
| `nextAction` | string | No | `agentic_response`, `run_handler` or `emit_event` (webhook only); see [Running a handler](#running-a-handler-nextaction-and-handler) |
| `handler` | object | No | `{ "name": "<service>" }` |

Only provided fields are updated. At least one updatable field is required. The trigger records `updatedBy` (the caller's
user id) and `updatedAt`. Writing the prompt field of the trigger's type removes a stray copy in the other field (a
`prompt` on an event trigger, a `promptTemplate` on a cron or webhook trigger). Any update also resets
`consecutiveFallbacks` and clears a `paused` outcome with its `lastError`.

A filter cannot be cleared through `update`; remove the trigger and add a plain one instead. On a filtered trigger
`event` is refused with `A filtered trigger lists its events in the filter, not in event`. The skill form is
`trigger.js update <id> --filter '<json>'` (or `--filter-file <path>`).

**Response:**

```json
{
  "success": true
}
```

---

## POST /v1/agents.triggers.remove

Remove a trigger from an agent.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `botId` | string | Yes | Agent bot user ID |
| `triggerId` | string | Yes | Trigger ID to remove |

**Response:**

```json
{
  "success": true
}
```

---

## POST /v1/agents.triggers.run

Manually fire a trigger immediately. Updates `lastRunAt` when the fire started, was handled, or was paused. A trigger with
`nextAction: run_handler` runs its handler first, exactly as a scheduled fire would.

**Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `botId` | string | Yes | Agent bot user ID |
| `triggerId` | string | Yes | Trigger ID to fire |

The trigger's prompt (or promptTemplate) is injected into the agent room with context `"Manual trigger run"`.

**Response:**

```json
{
  "success": true
}
```

**Errors:**

| Error | Cause |
|-------|-------|
| `Trigger not found` | Trigger ID doesn't exist on this bot |
| `Trigger has no prompt` | Trigger has neither prompt nor promptTemplate |
| `Failed to inject trigger message` | Agent room or bot user not found, nothing started, or the same trigger is already running (`skipped-active`) |

---

## POST /v1/agents.webhook/:token

Public endpoint for receiving external webhooks. No authentication — the unique token in the URL acts as the credential.

**URL Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `token` | string | Webhook trigger token (auto-generated) |

**Headers:**

| Header | Description |
|--------|-------------|
| `x-webhook-secret` | Shared secret (constant-time compare). |
| `Authorization: Bearer <secret>` | Alternate form, server accepts either. |

**Secret rules:**

- `emit_event` triggers — secret **mandatory**.
- `agentic_response` triggers — the secret is enforced whenever the trigger has one, which every trigger created through
  `add` does; a request without it answers 401.
- `run_handler` triggers — secret **mandatory**: a trigger without a stored secret answers 401, and a handler runs code on
  the posted body.

**Body:** Any JSON payload. The entire body is serialized and passed as context to the agent.

**Rate Limiting:**

60 requests per minute per agent (in-memory, resets every 60 seconds).

**Response (success):**

```json
{
  "success": true
}
```

**Response (errors):**

| Status | Cause |
|--------|-------|
| 404 | Token not found or trigger disabled |
| 401 | Secret missing or wrong |
| 429 | `error-too-many-requests` — rate limit exceeded |

**Example — agentic_response (no secret needed):**

```bash
curl -X POST https://chat.example.com/api/v1/agents.webhook/xyz789 \
  -H "Content-Type: application/json" \
  -d '{"from": "github", "event": "push", "repo": "my-app"}'
```

**Example — emit_event (secret required):**

```bash
curl -X POST https://chat.example.com/api/v1/agents.webhook/xyz789 \
  -H "Content-Type: application/json" \
  -H "x-webhook-secret: your-webhook-secret" \
  -d '{"from":"github","event":"push"}'
```
