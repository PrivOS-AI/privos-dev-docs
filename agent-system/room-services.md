# Room Services

A room service is a process an agent declares so it keeps running after the agent's turn ends: a listener or watcher
of an external service, a job longer than one turn, or a periodic poller. The runtime that runs the agent supervises
it. There is no inbound traffic to a service.

Background shells an agent starts inside a turn are killed when the turn ends (the attempt reaper). A service is not:
it is started, restarted, capped, logged and stopped by the runtime, and the room owner sees it in the board's
running-tasks bar with a stop button.

Hub events never need a service. Mentions, replies, DMs, notifications and busy rooms reach an agent through a
[filtered event trigger](./trigger-api-reference.md#subscription-filter) (`agent-scheduler`:
`trigger.js add --type event --filter '<json>'`); the hub listens for the agent.

A fourth kind, the **handler**, is not a long-lived process: it is a script a trigger runs on every fire to decide, without
a model turn, whether the agent is needed. See [Handlers](#handlers) and [Agent Routines](./agent-routines.md).

## Where services run

| Runtime | Supervisor | Credentials |
|---|---|---|
| Room-own sandbox (board mode: the board runs the agent itself) | the board process (`server.ts`) | **raw**: each binding named in `vault` is decrypted into the service env |
| Dedicated room VM | the VM core (`createApp`) | **relay**: relay tokens and relay URLs from `/run/privos-room/vault.env` |
| Collocated (shared) core | none | refused: `services need a dedicated room VM or a room-own sandbox (board mode)` |

A fleet tenant board in sandbox mode supervises nothing: it forwards the routes to the proxy, which forwards them to
the room's core.

The lane is a property of the runtime, not of a service. Board mode hands the raw value because the board already
runs as one OS user with its vault key file, its `vault.db` and the room bot key on the same disk, so a process running
as that user could already read all of them (the accepted same-uid ceiling). In a VM, keys never enter
the container; the relay injects them per request.

## Declaring services

`POST /api/projects/:id/services/apply` takes the **whole** declaration of a project; services left out are removed.
The runtime stores it in its own database (`service_definitions`), never in a file.

```json
{
  "version": 1,
  "services": [
    {
      "name": "telegram-watch",
      "kind": "daemon",
      "cmd": ["node", "watch.js"],
      "cwd": "AgentFiles/scripts/telegram-watch",
      "env": { "WAKE_WEBHOOK_URL": "https://<hub>/api/v1/agents.webhook/<token>", "WAKE_WEBHOOK_SECRET": "<secret>" },
      "vault": ["TELEGRAM_BOT_TOKEN"],
      "restart": "always",
      "maxMemoryMb": 256,
      "stopGraceSeconds": 5,
      "enabled": true
    }
  ]
}
```

| Field | Rule | Default |
|---|---|---|
| `name` | `^[a-z0-9][a-z0-9-]{0,39}$`, unique in the declaration | required |
| `kind` | `daemon`, `interval`, `oneshot` or `handler` | required |
| `cmd` | non-empty argv array of strings; never a shell string (use `["sh","-c","..."]` to ask for a shell) | required |
| `cwd` | relative to the project root; its real path (after symlinks) must be an existing directory inside the project | `.` |
| `env` | keys `^[A-Z][A-Z0-9_]*$`; not a platform-reserved name (`PRIVOS_`, `PROXY_`, `CLAUDE_`, `NODE_`, …); not a name also listed in `vault`; string values | `{}` |
| `vault` | binding env names the service needs | `[]` |
| `restart` | `always`, `on-failure`, `never`; on `interval` and `handler` only the default `never` is accepted (a listing echoes it back, so an apply built from a listing round-trips); a `oneshot` takes `on-failure` or `never` | `always` (daemon), `never` (oneshot, interval, handler) |
| `everySeconds` | `interval` only, integer ≥ 30 | required for `interval` |
| `timeoutSeconds` | `handler` only, integer 1–60: the run is ended after this many seconds | 30 |
| `maxMemoryMb` | integer ≥ 64 and ≤ `ROOM_SERVICES_MAX_MEMORY_MB` | the limit |
| `stopGraceSeconds` | integer 1–5 | 5 |
| `enabled` | boolean | `true` |

Any error refuses the whole apply and nothing is stored. A valid apply answers one result per service with the state
after the runtime tried to start it, so a refused start shows as `crashed_until_apply` with its reason. The response lists the errors, for example
`service odd: lane is not a known field` or `limit ROOM_SERVICES_MAX_PER_PROJECT=5 exceeded by service sixth`.

Listings show every env value as `"<set>"`. Sending `"<set>"` back in an apply keeps the stored value, so a client can
edit one service from a listing without knowing another service's secrets.

## Kinds

- **daemon**: runs continuously. On exit it restarts per `restart`, with a backoff of 1, 2, 4 … 60 s. Five runs in a
  row that each last under 60 s set the state `crashed_until_apply`.
- **interval**: runs `cmd` every `everySeconds`; a tick is skipped while the previous run is alive. It reuses the env
  of its first run until it is redeclared or restarted, so a vault binding is materialised once per start, not per
  tick.
- **oneshot**: runs once. On exit the state becomes `finished` with `exit <code>` as the reason; only a changed
  definition (or the owner's start) runs it again.
- **handler**: no process between runs. It runs only when asked, with a JSON payload on stdin, for at most
  `timeoutSeconds`; see [Handlers](#handlers).

The newest 20 finished instance rows of each service are kept. Every run is a `shells` row with `kind = 'service'`, `service_name`, `pgid`, `started_at` (the `/proc` start time) and
`attempt_id = NULL`, so the attempt reaper and the shell restore loops never touch it. The process runs in its own
process group with stdout and stderr appended to `<data root>/services/<projectId>/<name>.log` (kept to
`ROOM_SERVICES_LOG_MAX_MB`, one previous file `.log.1`).

## States and reconcile

| State | Set by | Started by reconcile |
|---|---|---|
| `enabled` | apply, the owner's start | yes |
| `on_demand` | apply and the owner's start, for a **handler** (never `enabled`) | never: it runs only when called |
| `owner_disabled` | a person's stop from the board | never; an apply reports `owner_disabled` and changes nothing |
| `crashed_until_apply` | the short-run ceiling, a refused start, a revoked binding | no; the next apply or the owner's start clears it |
| `finished` | a oneshot's exit, a daemon with `restart: never` that exited | no; a changed definition or the owner's start |

Reconcile compares definitions with running instances and starts, stops or restarts to match, in one queue per
project. It runs when the runtime boots, after every agent attempt, and on apply. A definition whose hash changed is
restarted.

**The owner's stop is final.** A stop from the board (the shared key or a person's key) sets `owner_disabled`; only a
person's start clears it, and an agent's apply can neither restart, change nor remove that service. A stop from the
agent (`service.js stop`, or any request with `x-privos-service-caller: agent`) only ends the current run.

Stops send `SIGTERM` to the process group and `SIGKILL` after `stopGraceSeconds`. Before signalling, the runtime checks
that the pid still has the start time it recorded; a recycled pid is never signalled (the row ends with
`exit_signal = 'lost'`). On boot, an instance left by the previous server is killed and confirmed gone, then reconcile
starts exactly one new instance. On shutdown, every service gets its grace before the server exits.

Memory is checked every 15 s over the whole process group; above `maxMemoryMb` the group is killed (`exit_signal =
'memory'`) and the restart policy applies.

## Handlers

A handler is declared like any service (`kind: handler`, a `cmd`, optional `cwd`, `env`, `vault`, `timeoutSeconds`,
`maxMemoryMb`, `stopGraceSeconds`) and stored in the state `on_demand`. Nothing starts it: not boot, not reconcile, not an
older supervisor that starts only `enabled` services. The hub or the agent calls it, and it counts toward
`ROOM_SERVICES_MAX_PER_PROJECT`. What the exit code means to a trigger, and why the hub runs the model turn when a handler
fails, is in [Agent Routines](./agent-routines.md#the-check-stage-handlers).

A handler belongs to a project. The hub looks for a trigger's handler in the **agent room's** project (the agent room id,
`agent-room-<botId>` for current agents), so declare it from a turn in the agent room.

### The `run` route

`POST /api/projects/:id/services/:name/run` runs the handler once. The body is the stdin payload: one JSON object, at most
256 KB (an absent body is `{}`).

| Header | Meaning |
|---|---|
| `x-privos-run-deadline` | Epoch milliseconds by which the caller stops waiting. Optional. The run starts only if its `timeoutSeconds` plus `stopGraceSeconds` fit before it |
| `x-privos-service-caller` | `agent` marks the in-project agent. The caller class itself comes from the credential (board key or a proxied caller is the platform, loopback is the agent); `platform-run`, which the hub sends, changes nothing on any other route |

Callers: the platform (the hub) and the in-project agent, which dry-runs its own handlers (`service.js run`). The platform
never queues: while a run of that handler is in flight it is refused at once with `handler-busy`. The agent may wait behind
one run (queue depth 1). Both re-check state and deadline when they reach the front.

**Answers.** Errors have the shape `{ "error": "<code>[: <reason>]", "code": "<code>", "reason"?: "…" }`.

| Status | `code` | Cause |
|---|---|---|
| 200 | none | The run finished; the body is the result below. A timeout is still a 200 with `timedOut: true` |
| 400 | `invalid-payload` | The body is not one JSON object (`the body must be one JSON object`); a JSON `null` or an array is refused |
| 400 | `payload-too-large` | Over 256 KB (`the payload is larger than 256 KB`), also from the declared size, before the body is read |
| 400 | `not-a-handler` | The service exists with another kind |
| 400 | n/a | `invalid service name` |
| 404 | `unknown-service` | No such service |
| 409 | `owner-disabled` | A person stopped the handler; only a person can start it |
| 409 | `handler-busy` | A platform call while a run is in flight (or a second waiting agent) |
| 409 | `not-startable` | `reason` says why: `crashed_until_apply` reasons, `vault-binding-missing: <NAME>`, `vault-raw-export-…`, `disabled`, `project-not-found`, `cwd-not-found: <cwd>`, `stopped`, `shutting-down`, `spawn-failed`, `start-failed` |
| 409 | `deadline-passed` | The time left before `x-privos-run-deadline` is shorter than `timeoutSeconds` plus `stopGraceSeconds` |
| 409 | `caller-gone` | The caller disconnected before the run started |
| 503 | n/a | `room services are not running on this runtime` |

**Result (200):**

```json
{ "exitCode": 0, "signal": null, "timedOut": false, "durationMs": 412, "stdout": "tail up to 8 KB", "stderr": "tail up to 2 KB", "truncated": false, "stopReason": "owner-stopped" }
```

`stopReason` is present only when the platform ended the run, never for a natural exit or a timeout: `owner-stopped`,
`stopped`, `redeclared`, `vault-binding-revoked`, `vault-binding-rotated`, `project-deleted`, `shutdown`, `memory`,
`caller-gone`, `lost` (the leader survived SIGKILL; `exitCode` is `null` and `signal` is `SIGKILL`). `truncated` is true when
the 64 KB capture or an 8 KB / 2 KB tail dropped output.

**Deadline semantics.** The deadline is a must-finish-by time: a run is refused rather than started if its own timeout and
grace would carry it past the caller's patience. The hub sends now + 120 s. A run whose caller disconnects is killed
(`stopReason: caller-gone`). `timeoutSeconds` sends SIGTERM to the whole process group and SIGKILL after
`stopGraceSeconds`; after the leader exits the rest of the group is killed too, so nothing a handler backgrounds survives.

**Proxy budget.** In a VM the request crosses the proxy, which aborts every other forwarded route at 30 s with a 502. The
`run` route gets 90 s (`SERVICE_RUN_TIMEOUT_MS` in `packages/proxy/src/routes/forward.ts`) and marks the project active, so a
running handler keeps its core from idling out.

**Capture and logs.** Both pipes are drained continuously and capped at 64 KB each; no output reaches board sockets. Every
run is a `shells` row (`kind = 'service'`) and appends `[run <shellId> exit <code> <ms>ms]` followed by `[stdout]` and
`[stderr]` blocks to the service log. On the board, any raw credential value the handler received is replaced by the mask in
the returned output and the log; if masking fails, the output is replaced by `[output withheld: masking failed]`.

**Listing.** For a handler, `GET /api/projects/:id/services` adds `kind`, `timeoutSeconds` and
`lastRun: { at, exitCode, durationMs, timedOut }` (null before the first run). `/health` counts a handler in
`services.running` only while a run is in flight. The running-tasks bar shows the kind `handler` and its last run in the
place where other services show a pid; stop and start work as for other services.

**Credentials.** The same lanes as other services. On the board the env is built once per activation (one `materialized` use
row per binding), reused across runs, and dropped on redeclare, rotation or revoke; a revoke sets `crashed_until_apply` with
`vault-binding-revoked: <NAME>`.

## Credentials

- **Board mode (raw):** before the process starts, the vault writes one `credential_use_log` row per binding
  (`outcome = 'materialized'`, `method = 'SERVICE'`, fingerprint only) and one `credential.materialize` lifecycle entry
  under the binding's scope, which the vault audit viewer lists. If a write fails, the service does not start. The
  value goes only into the child's environment: never into the relay env file, a log, a route response or the board's
  own environment. The base-URL variable holds the real `https://<host><prefix>`. Bindings in the tenant-global or a
  universal-bot scope, on the hub host or on a `privos.io` host are never exported raw
  (`vault-raw-export-refused: <NAME>`).
- **VM (relay):** the service gets the core env minus board-only names, then `skills.env`, then `vault.env`, then its
  declared env; vault names are applied last, so a declaration cannot replace a relay name. Call APIs through the
  binding's base-URL variable.
- A named binding the project does not export refuses the start: `vault-binding-missing: <NAME>`. Refusals a retry
  cannot fix (a missing binding, a refused binding, a missing `cwd`, a command that cannot be executed) park the service
  in `crashed_until_apply`; others (a closed vault, a busy workspace lease, a database error) are retried with the
  crash backoff.
- **Rotate and revoke:** on the board, rotating a binding restarts the services that use it (a new materialisation,
  new audit rows) and revoking it stops them (`crashed_until_apply`, `vault-binding-revoked: <NAME>`). In a VM the relay
  resolves every request, so a rotation needs nothing and a revocation is felt as a refusal at the service's next call.

## Waking the agent

The supervisor never calls the hub. A service that needs its agent calls the agent's own webhook trigger: create one
with `trigger.js add --type webhook --next-action agentic_response`, and put the printed URL and secret in the
service `env` as `WAKE_WEBHOOK_URL` and `WAKE_WEBHOOK_SECRET`. The runtime stores them and redacts them everywhere. The
service posts to the URL with the `X-Webhook-Secret` header only when something needs the agent.

## Limits

Per project, from the runtime's own environment (never from the agent-writable settings route):

| Variable | Default |
|---|---|
| `ROOM_SERVICES_MAX_PER_PROJECT` | 5 |
| `ROOM_SERVICES_MAX_MEMORY_MB` | 512 (per service, over its process group) |
| `ROOM_SERVICES_LOG_MAX_MB` | 10 (per service) |

The board reads its own env; a core gets them from the proxy env. Fleet tenants set them through the portal as
`TENANT_ROOM_SERVICES_*`.

## Routes

| Route | Who |
|---|---|
| `GET /api/projects/:id/services` | definitions (env redacted), states, running pid, restarts, last instances |
| `POST /api/projects/:id/services/apply` | the agent or a person |
| `DELETE /api/projects/:id/services/:name` | a person's stop is final; an agent's ends the current run |
| `POST /api/projects/:id/services/:name/enable` | a person only |
| `GET /api/projects/:id/services/:name/logs?lines=200` | log tail |
| `POST /api/projects/:id/services/:name/run` | the platform or the in-project agent: run a handler once, see [The `run` route](#the-run-route) |
| `POST /api/projects/:id/services/stop-all` | a person or the platform; `?ifIdle=1` refuses (409) while an attempt, a plain shell or a terminal runs; the proxy uses it to move a core to a new runtime contract |
| `POST /api/projects/:id/services/reconcile` | a person or the platform: start what should run (the proxy calls it when a recreate could not follow a stop) |

On the board, a person is the shared key or a person's ring key; an agent ring key is not, and any caller sending
`x-privos-service-caller: agent` is treated as the agent. In a VM core, a loopback caller is the in-VM agent and a
caller forwarded by the proxy is the platform; a call from the container's own addresses (loopback or its network
address) is the agent. Deleting a project stops its services and drops their definitions
first.

## The `privos-services` skill

Agents use `node $SKAWLD_SKILL_DIR/service.js` (`list`, `status`, `logs`, `add`, `remove`, `stop`, `apply --file`, `run`).
`add` edits the listed declaration and applies it; env values are never printed. `add --kind handler --timeout-seconds N`
declares a handler; `run <name> --payload-file payload.json` dry-runs it once and prints the exit code, duration,
`timedOut` and the stdout tail (`stopped by the platform (<reason>)` when the platform ended it). On the board the skill calls
`http://127.0.0.1:$PORT` with `API_ACCESS_KEY`; in a VM it calls the core at `http://127.0.0.1:$PORT`. Both always
send `x-privos-service-caller: agent`.

Rules the skill and `CLAUDE.md` give agents: never `nohup`, `&`, `setsid` or `pm2` a long-running process; never ask
for a human's personal token; hub events go through filtered triggers.

## Related docs

- [Credential Vault](./credential-vault.md)
- [Trigger API Reference](./trigger-api-reference.md)
- [Agent Routines](./agent-routines.md)
- [Super Agent](./super-agent.md)
