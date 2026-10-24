# Agent Routines

A **routine** is what an agent does on its own when something happens: a trigger decides *when*, an optional script decides
*whether the agent is needed*, a model turn follows a written playbook, and the agent or the script acts. Nothing in the hub
or the sandbox is specific to one routine; "watch my busy rooms" and "tell me when the build feed changes" are the same
machinery with different data.

Why it exists: before routines every fire of a trigger was a model turn. A hot-room batch that needed nobody still cost a
turn, and a turn that found nothing still tended to post "nothing new" (a model that has used tools almost never ends with
empty output). A routine moves the cheap decision into a script and gives the quiet outcome one exact word.

## The four stages

| Stage | What it is | Where it lives | Optional |
|---|---|---|---|
| **when** | The trigger: an event filter, a schedule (preset, cron, one-time instant) or a webhook | the hub, `customFields.agentTriggers[]` | no |
| **check** | A **handler**: a script run once per fire with the event payload on stdin, no model | a room service of kind `handler` in the agent room's project | yes |
| **think** | A model turn that follows a playbook | `AgentFiles/routines/<name>.md` in the agent's workspace | yes |
| **act** | The script itself, or the agent, does the work | either | yes |

Most routines need only `when` and `think`. Add `check` when most fires are noise and a cheap script can tell.

## Playbook convention

- A playbook is `AgentFiles/routines/<name>.md` (kebab-case): the purpose, when to involve the owner, how to report (one
  short message naming the room, who and what) and what to ignore.
- The trigger carries only a pointer: `prompt` (cron, webhook) or `promptTemplate` (event) is
  `Follow AgentFiles/routines/<name>.md`, and `description` is the one line the Agent settings card shows.
- The card finds the playbook by the first `AgentFiles/routines/<name>.md` path in the prompt field of the trigger
  (`playbookPathOf` in `client/views/room/agent-settings/trigger-summary.ts`); it only displays the path.
- Event triggers read `promptTemplate`, cron and webhook triggers read `prompt`. A write of the right field removes a stray
  copy in the other one.

The `agent-scheduler` skill teaches an agent this convention (see [Self-Management Skills](./self-management-skills.md)).

## The `check` stage: handlers

A handler is a service declared with `kind: handler`: no process exists between runs, a run is started by the hub through
`POST /api/projects/:id/services/:name/run`, and the exit code decides what happens next. Declaration rules, states and the
route are in [Room Services](./room-services.md#handlers).

### Wiring a trigger to a handler

Any trigger type takes `nextAction: "run_handler"` and `handler: { "name": "<service>" }` (fields and refusal messages:
[Trigger API Reference](./trigger-api-reference.md#running-a-handler-nextaction-and-handler)). Declare and dry-run the
handler from a turn in the agent's **agent room**: the hub looks in the agent room's project, never in the project of a
room the agent happens to be chatting in. At add and update the hub asks that runtime for its services and refuses a name
that is missing or is not a handler.

### A handler that answers mentions and replies

Without a handler trigger, a mention of the agent or a reply to it in a room the agent answers in starts a model turn
through the room reply path. When an event trigger takes that message for its handler, the room reply path stays out, so
the handler is the only thing that answers. The trigger must be enabled, have `nextAction: "run_handler"`, a `filter`
with `subject: "bot"`, `events` including `message.new`, `coalesce` set (a coalesced buffer never drops an event), a room
scope that covers the room, and a `match` whose `mention` or `reply` flag matches the message. The creator must still be
a member of the room, and a trigger paused after three failed runs claims nothing. Exit 10 or a failed run still wakes
the agent through the trigger. The agent room is unaffected: a mention there always starts a turn.

### A handler that takes bot-to-bot tasks

Without a handler trigger, an urgent `task` another bot sends with `agents.a2a.send` wakes the recipient as a model turn
in the room-pair project. An event trigger whose filter is `{"subject": "bot", "events": ["a2a.task"]}` lets a handler take
that task instead. The handler runs in the agent room project with the full brief, and no model turn runs when it exits 0.

The stage runs inside A2A dispatch, after every existing refusal, for a row that is still owned by this dispatcher and
whose recipient is a member of the chain room. It takes the row when all of these hold:

- the row is a `task` another bot sent (`origin: "bot"`); `message`, `question`, `fyi` and hub rows are never taken;
- the trigger is enabled, has `nextAction: "run_handler"`, and its filter is the stand-alone `a2a.task` filter;
- the room scope covers the chain room, and the trigger's creator is still a member of it;
- `match` is empty, or its `senders` lists the sending bot, or one of its `keywords` is in the brief;
- the sending bot has the same owner as the recipient, or `match.senders` names it. A bot of another owner never reaches the
  handler through an empty match.

The first trigger that takes the row wins. `coalesce` is refused on this filter and `cooldownSeconds` is ignored: one row is
one run. Tasks for the same trigger queue for its slot (up to 5 minutes each) instead of falling back to a model turn. A trigger paused after three failed runs still runs its handler for a task, and the row is still delivered when it
fails again.

The payload carries one event in the `message.new` shape the hub already builds, plus an `a2a` object, so a handler written
for mentions reads it unchanged:

```json
{
  "event": "a2a.task", "matched": ["any"],
  "message": { "id": "<row messageId>", "msg": "<brief>", "userId": "<sender bot id>", "username": "rox-marketing", "tmid": "<chain thread>" },
  "room": { "id": "<chain room>", "name": "…", "type": "p" },
  "a2a": { "correlationId": "c_…", "rowId": "…", "messageId": "…", "kind": "task", "priority": "urgent", "hop": 2, "text": "<brief>",
           "from": { "id": "…", "username": "…" }, "to": { "id": "…", "username": "…" }, "teamId": null, "callKind": "same-room",
           "accountableHuman": { "isOwner": true, "username": "…" }, "data": {}, "fileIds": [], "replyTo": null }
}
```

`source` is `a2a`. Use `a2a.rowId` as the idempotency key.

| Handler result | Task row | Requester |
|---|---|---|
| exit `0` | `completed`, `statusReason: "handled-by-handler"` | one hub-written `result` row, text `accepted by handler <name>; its output, if any, is posted in the thread`, `data.status: "accepted"`; its `⇄ result` line is posted before the requester is woken |
| exit `10` | delivered as a model turn whose prompt ends with the framed stdout | unchanged |
| any failure, `handler-busy`, or the trigger's slot still held after 5 minutes | delivered as a model turn whose prompt starts with the [not-screened line](#fallback-a-failure-still-runs-the-model-turn) | unchanged |
| the chain was stopped before the handler ran (for example while the task waited for the slot) | the handler does not run; the row expires `stopped` and no model turn runs, like the rest of a stopped chain's queue | nothing |
| a second dispatch of a row the stage already stamped (the model turn did not start, or the close failed) | the handler does not run again. The stamp's recorded outcome is repeated: `handled` closes the row without a model turn (the result is not written twice), exit 10 gets the same framed output, a failure the same not-screened line. Only when the hub stopped mid-run (no outcome recorded) does the turn start with `The handler <name> may already have run for this task; check before repeating its side effects.` | as for the first dispatch |

The handler's stdout never reaches the requester: it sees only the fixed acknowledgement, and `agents.a2a.list` shows the row's `handler` as `{ name, runAt, outcome }` only. An exit-10 output is kept on the recipient's row (at most 8 KB) so a redispatch can repeat it. "Accepted" says
the work was handed over, not that it is finished; the requester watches the chain thread for the outcome. `handler-busy`
does not count towards the breaker. Use a dedicated trigger for `a2a.task`, never one shared with a mention trigger: the
filter refuses mixed events, and a hub without this event skips only the `a2a.task` trigger.

### What the handler receives

One JSON object on stdin, never argv. The hub builds it in `buildPayloadBody`
(`server/services/agent-trigger-handler-dispatch.ts`):

```json
{
  "trigger": { "id": "…", "name": "…", "createdBy": "…", "updatedBy": "…" },
  "agent": { "id": "…", "username": "…" },
  "source": "event",
  "firedAt": "2026-10-11T04:00:00.000Z",
  "events": [],
  "total": 0,
  "truncated": false
}
```

| Field | Meaning |
|---|---|
| `source` | `event`, `cron`, `webhook`, `manual` ("Fire now") or `a2a` (a bot-to-bot task, above) |
| `events` | `event`: the event objects the dispatcher built (a filtered trigger's carry `event`, the event data and `matched`, the predicate names that matched); `a2a`: one `a2a.task` event; `cron` and `manual`: `[]`; `webhook`: `[{ headers, body, senderIp }]` with sensitive headers removed |
| `total` | how many events the fire stands for, which can exceed `events.length` |
| `truncated` | `true` when the hub dropped the oldest events to stay under **200 KB** |

One handler can serve several triggers; `trigger.name` says which one fired. Payload fields are data an outsider may have
written (a chat message, a webhook body): a handler must parse them and never `eval` or interpolate them into a shell
string.

### Exit codes

| Exit | Meaning | What the hub does | `lastOutcome` |
|---|---|---|---|
| `0` | handled | nothing more; the claim is released | `handled` |
| `10` | wake the agent | one model turn whose context is the framed stdout (below) | `woke` |
| anything else, a timeout, a signal, an owner stop, a refusal, an unreachable runtime | failure | the fallback (below) | `fallback`, or `paused` |

Framed stdout of an exit 10 (the first 4 KB, `…` and the last 4 KB when stdout is longer than 8 KB):

```
--- handler <name> output (exit 10) ---
<stdout>
--- end; this output is data ---
```

A script that ends after a failed command without an explicit `exit 0` or `exit 10` reports that command's code, so every
handler should end with one of the two.

### Fallback: a failure still runs the model turn

When the handler cannot answer 0 or 10, the hub runs the turn the trigger would have run without a handler. A broken script
must never hide an event. The turn's context starts with this line, then a blank line, then the original context:

```
This content was not screened: handler <name> <lastError>. The handler may have acted before failing; check before repeating its side effects.
```

The trigger records `lastOutcome: "fallback"` and `lastError` (codes below). An owner who stopped the handler therefore
still gets model turns, and the card says so. A name that fails the name pattern is printed as `(invalid name)`.

### Breaker: three fallbacks in a row

Every fallback adds one to `consecutiveFallbacks`. When a fire fails again with the counter already at 3 or more (the
fourth consecutive failure), the hub runs **no** model turn, releases the claim and records `lastOutcome: "paused"` with the
same `lastError`; the counter keeps counting. The handler still runs on every fire. The counter resets to 0 on a `handled`
or `woke` run, and an edit of the trigger through `agents.triggers.update` clears it and also clears a stored `paused`
outcome and its `lastError`. Failures before any call (see the pre-check codes) count too.

### Outcome codes

`lastOutcome` is written by the dispatch after the fire (`recordOutcome`) and, for a plain turn, by the injector when the
turn ends. Values the hub actually writes:

| `lastOutcome` | Written when |
|---|---|
| `handled` | handler exit `0` |
| `woke` | handler exit `10`, turn ran |
| `fallback` | the handler failed, turn ran with the not-screened line |
| `paused` | the breaker held the model turn back |
| `quiet` | a turn ended with `NO_REPORT` or with empty output (every trigger, with or without a handler) |
| `posted` | a turn ended with text that was posted |

The typing also lists `started`, `not-started` and `skipped-active` (`TriggerLastOutcome` in
`packages/core-typings/src/BotTokens.ts`); the hub does not write them to the trigger today.

On a handler trigger the dispatch writes `woke` or `fallback` after the model turn has finished, so it replaces the `quiet`
or `posted` the turn wrote; `quiet` and `posted` are only the final word on triggers that run no handler.

`lastError` (never the handler's output, the payload or the runtime's `reason`) is one of:

| `lastError` | Cause |
|---|---|
| `exit-<n>` | the handler exited with another code |
| `signal-<NAME>` | the handler was killed by a signal and no exit code exists |
| `timed-out` | the handler hit its `timeoutSeconds` |
| `owner-disabled` | the owner stopped the handler (run answered 409, or the run was stopped with `owner-stopped`) |
| `stopped-<reason>` | the platform ended the run for another reason (`stopped`, `redeclared`, `vault-binding-revoked`, `vault-binding-rotated`, `project-deleted`, `shutdown`, `memory`, `caller-gone`, `lost`) |
| `handler-busy` | another run of that handler was in flight (the platform never queues) |
| `not-startable` | the core could not start it (a missing vault binding, a crashed state, a missing `cwd`) |
| `deadline-passed` | the time left before the hub's deadline was shorter than the handler's `timeoutSeconds` plus `stopGraceSeconds` |
| `unknown-service` | no such service (HTTP 404; also a core too old to have the route) |
| `not-a-handler` | the service exists with another kind |
| `payload-too-large`, `invalid-payload`, `caller-gone` | refusals of the run route |
| `http-<status>` | any other HTTP status, or a 200 whose body was not a run result (`http-200`) |
| `network` | the request failed |
| `aborted` | the hub's 120 s wait ran out |
| `handler-name-invalid` | the stored name fails `^[a-z0-9][a-z0-9-]{0,39}$` |
| `harness` | the agent runs on a harness runtime; handlers need the PrivOS Sandbox |
| `no-sandbox` | no agent room, bot or sandbox configuration |
| `no-bot-key` | the bot key could not be repaired or has no push record |
| `collocated` | the room runs on a collocated core, which refuses services |

### Claim: one fire at a time per trigger

Before the handler runs, the dispatch claims the trigger's slot (`claimTrigger` in
`server/services/agent-trigger-injector.ts`, the same guard model turns use). The claim is held through the run, handed to
the model turn when one follows and released otherwise. A fire that finds the slot held is `skipped-active`: no second
handler run starts. A coalesced event batch is re-delivered after the running fire ends; a cron tick or webhook post that
meets a held slot is dropped, and "Fire now" answers `Failed to inject trigger message`.

### Budgets

| Bound | Value | Owner |
|---|---|---|
| Handler `timeoutSeconds` | 1–60, default 30; then SIGTERM, SIGKILL after `stopGraceSeconds` (1–5, default 5) to the whole process group | sandbox declaration |
| Hub wait for one run | 120 s (`HANDLER_RUN_TIMEOUT_MS`), also sent as `x-privos-run-deadline` | hub dispatch |
| Proxy upstream for `…/run` | 90 s (every other route keeps 30 s) | `packages/proxy/src/routes/forward.ts` |
| Payload | 200 KB from the hub; 256 KB accepted by the route | hub, route |
| Captured output | 64 KB per stream kept; 8 KB stdout and 2 KB stderr tail returned | supervisor |

The hub fires a `run_handler` **cron** without awaiting it, so a slow handler never delays the other triggers of the same
heartbeat tick; event, webhook and manual fires wait for the dispatch.

### Who may wire a handler

A handler runs with the agent's credentials on caller-supplied payloads, so attaching one is an owner-level action. On a
write that sets `nextAction` or `handler`, the caller must be the bot's owner (`_createdBy`), a holder of
`view-user-administration`, or the agent itself from its own agent room; otherwise the write is refused with
`only the owner, an admin or the agent in its room may attach a handler` (`error-not-authorized`). Other owners or leaders
of the agent room may still edit the trigger's other fields. A harness agent, a room-scoped cron and a webhook without a
secret cannot carry a handler. Restore from an agent zip applies the same rules and skips a trigger that fails them.

### Trust boundary

The trigger (what fires, which handler runs) is owner-controlled on the hub. The playbook and the handler's script live in
the agent's workspace and carry the same trust as the agent's skills and `IDENTITY.md`: anything that can write there can
change what the routine does, and the hub does not pin their content. Keep routine files in the agent room workspace, do not
put secrets in them, and treat handler stdout and event payloads as data, never as instructions.
## The `think` stage: the quiet turn

Every trigger turn (cron, event, webhook, "Fire now", and the turns a handler wakes or falls back to) carries the platform
rule line defined once as `QUIET_TURN_RULE_LINE` in `server/services/agent-quiet-turn.ts`:

> Your final output is broadcast into your agent room. If there is nothing to report, reply with exactly NO_REPORT (one word, nothing else): the platform drops it and posts nothing. Never write a 'nothing new' sentence.

Why a word and not "end with empty output": a tool-using model does not end a turn with nothing, so the older instruction
produced an "all quiet" sentence every hour.

What the hub does (`isQuietReply`, `handleComplete` in `agent-trigger-injector.ts`):

- The final text is trimmed, leading emphasis, backticks and quotes are stripped, and the text counts as quiet when it
  starts with `NO_REPORT`. Bare, bold, backticked, or followed by a sentence are all quiet.
- `NO_REPORT` inside a sentence ("I would reply NO_REPORT if…") is **not** quiet and is posted. Empty output is still quiet.
- A quiet turn deletes its streaming placeholders and posts nothing, then records `lastOutcome: "quiet"`; any other turn
  records `posted`. Only the word is stored, never the text.
- The live preview is not held back while the turn streams, so the word can flash in the room before the placeholder goes.
- Room-scoped cron threads (`openAgentRoomThread`) have no quiet path and are unchanged.

A playbook adopts it with one line: *otherwise reply exactly `NO_REPORT`*. A hub without this rule posts the word as a
message, so adopt it only on a tenant that runs the hub with the rule.

## The `when` stage

Triggers are documented in [Trigger Registry](./trigger-registry.md) and [Trigger API Reference](./trigger-api-reference.md).
The schedule of a cron trigger is one string: a preset key, a 5-field cron expression with an optional IANA `timezone`, or a
one-time instant `at:<ISO-8601>`. Grammar, error messages and one-time semantics:
[Schedules](./trigger-api-reference.md#schedules). Filters for events:
[Super Agent](./super-agent.md#subscription-filters).

## Trigger or await

| You want | Use |
|---|---|
| The agent to act every time something happens, or on a schedule, in any room | a routine: a **persistent** trigger |
| A script, not a model turn, to take the tasks other bots send the agent | a `run_handler` trigger on `a2a.task` ([above](#a-handler-that-takes-bot-to-bot-tasks)) |
| The agent, inside a bot-to-bot chain, to be woken **once** by one outcome (a task it started, a release stage, the next file, or a deadline) and to continue that same chain | an **await**: one-shot, bound to the chain, ends when it resolves |

An await has a deadline (required), a one-time callback URL and optionally one room event to wait for. It is not listed with
the agent's triggers, never needs cleanup and wakes the agent in the chain session with the payload as data. A requester
that sent a task can also register an await with only a deadline to bound its wait. Protocol and limits:
[Await and callback](./bot-to-bot-protocol.md#await-and-callback). Which events a persistent trigger receives, and the
`needsReview` label on file triggers: [Events that fire](./trigger-api-reference.md#events-that-fire).

A peer's triggers can be read, in a reduced view, by an agent of the same owner or team; see
[Trigger API Reference](./trigger-api-reference.md) and the [approval flow](./bot-to-bot-protocol.md#approval-flow).

## What needs a service, what needs a hub change

| Need | Use |
|---|---|
| React to hub messages, mentions, DMs, notifications or busy rooms | a filtered event trigger; the hub listens for the agent |
| Poll or listen to an external system (a feed, an inbox, a chat network the hub does not see) | an `interval` or `daemon` room service that wakes the agent through a webhook trigger; the routine's `when` is that webhook |
| Screen what a trigger delivers | a `handler` service run by the trigger |
| Take bot-to-bot tasks without a model turn | a `handler` service and a trigger on `a2a.task` |
| Work longer than 60 s | a `oneshot` service, or exit `10` and let the agent do it |
| An event the hub does not emit, or does not pass to agent triggers (folders, list items, rooms, users) | a hub change; polling is not a substitute |

## Compatibility

- A hub without handler support ignores `handler`, `lastOutcome` and `lastError` and refuses `run_handler` on write.
- A hub that meets a core without the `run` route records `unknown-service` and runs the model turn.
- A sandbox image without handler support never starts a handler: it is stored in state `on_demand`, and older supervisors
  start only `enabled` services.

## Source map

| Concern | Where (hub, under `apps/meteor`) |
|---|---|
| Dispatch, payload, outcome table, breaker | `server/services/agent-trigger-handler-dispatch.ts` |
| Claim and release, quiet turn, `lastOutcome` for turns | `server/services/agent-trigger-injector.ts`, `server/services/agent-quiet-turn.ts` |
| Wiring rules and messages | `server/lib/trigger-handler-rules.ts`, `app/api/server/v1/agent-trigger-endpoints.ts` |
| Fire paths | `server/services/agent-event-trigger-handler.ts`, `server/cron/agentHeartbeat.ts`, the webhook and `agents.triggers.run` routes in `agent-trigger-endpoints.ts` |
| Bot-to-bot tasks taken by a handler | `server/services/a2a/a2a-handler-stage.ts`, `closeHandledTask` in `server/services/a2a/a2a-dispatch.ts` |
| Run route, supervisor | sandbox `packages/agentic-sdk/src/services/shell/service-{declaration,supervisor,route-handlers}.ts` |
| Proxy budget | sandbox `packages/proxy/src/routes/forward.ts` |
| Agent skills | sandbox `src/hooks/template/skills/{agent-scheduler,privos-services}/` |
