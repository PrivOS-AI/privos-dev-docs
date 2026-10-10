# Self-Management Skills

Agents can manage their own triggers via natural language. When a user asks an agent to "check for news every hour", the agent reads its skill files and uses `tool_use` (Claude's tool calling) to execute curl/node commands against the trigger API.

## How It Works

```
User: "Set up a cron job to check news every hour"
        │
        ▼
Agent (PrivOS Sandbox / Claude Code)
        │
        ├─ Reads .claude/skills/privos-agent-management/SKILL.md
        ├─ Understands available trigger management scripts
        │
        ▼
tool_use: bash
  node .claude/skills/privos-agent-management/trigger-add.js cron every_1h "Check trending news"
        │
        ▼
Script reads .env, calls POST /v1/agents.triggers.add
        │
        ▼
Agent confirms: "Done! I'll check trending news every hour."
```

## Skill Files

Uploaded to MinIO during agent creation, synced to PrivOS Sandbox CWD via `projectId = roomId`.

### Directory Structure

```
.claude/skills/privos-agent-management/
├── .env              — bot credentials
├── SKILL.md          — usage documentation
├── trigger-list.js   — list all triggers
├─�� trigger-add.js    — add a new trigger
├��─ trigger-update.js — update an existing trigger
├── trigger-remove.js — remove a trigger
└── trigger-run.js    — manually fire a trigger
```

### .env

```
AGENT_BOT_ID={botId}
AGENT_BOT_TOKEN={botToken}
PRIVOS_HUB_URL={ROOT_URL}
```

### SKILL.md

Documents all scripts with usage examples. The agent reads this to understand what commands are available.

### Script Pattern

All scripts share the same env-loading pattern:

```js
const { readFileSync } = require('fs');
const { join } = require('path');

const env = Object.fromEntries(
  readFileSync(join(__dirname, '.env'), 'utf-8')
    .split('\n').filter(Boolean).map(l => l.split('=').map(s => s.trim()))
);
```

Then call the appropriate API endpoint using `fetch()` with `X-Auth-Token: env.AGENT_BOT_TOKEN`.

## Script Reference

### trigger-list.js

```bash
node .claude/skills/privos-agent-management/trigger-list.js
```

Lists all triggers with status, type, schedule/event, prompt, and last run time.

Output format:
```
1. ☑ [cron] every_1h — "Check trending news" (last: 2026-04-16T10:00:00.000Z) id=abc123
2. ☐ [webhook] webhook — "Process email" (never run) id=def456
```

### trigger-add.js

```bash
# Cron
node trigger-add.js cron every_1h "Check trending news"

# Webhook
node trigger-add.js webhook "Process incoming email"

# Event
node trigger-add.js event message.new ROOM_ID "Triage this message"
```

Output:
```
Trigger added: [cron] id=abc123
```

For webhooks, also prints:
```
Webhook URL: https://chat.example.com/api/v1/agents.webhook/xyz789
```

### trigger-update.js

```bash
# Change schedule
node trigger-update.js TRIGGER_ID schedule=every_30m

# Change prompt
node trigger-update.js TRIGGER_ID prompt="New prompt text"

# Toggle enabled/disabled
node trigger-update.js TRIGGER_ID enabled=false

# Multiple fields
node trigger-update.js TRIGGER_ID schedule=every_6h prompt="Updated task"
```

### trigger-remove.js

```bash
node trigger-remove.js TRIGGER_ID
```

### trigger-run.js

Manually fire a trigger immediately.

```bash
node trigger-run.js TRIGGER_ID
```

## Skills that ship with the sandbox

Two skills come from the PrivOS Sandbox image (`src/hooks/template/skills/` in the sandbox repository) and are what an agent
uses for routines. Their `SKILL.md` is the agent-facing contract; this section records the flags and where they meet the hub.

### `agent-scheduler` (`trigger.js`)

`node ${SKAWLD_SKILL_DIR}/trigger.js <list|add|update|remove|run|regenerate-secret|explain> [flags]`; `--json` on any
subcommand. It manages only the calling agent's own triggers and always asks the user before a write.

**Schedule flags (cron triggers).** Exactly one family per call, otherwise
`Pass only one of --schedule, --once, --in, or --time/--weekly.` Every flag produces a schedule the hub accepts
([Schedules](./trigger-api-reference.md#schedules)); the CLI computes every instant from its own clock, so the model never
does time arithmetic.

| Flag | Result |
|---|---|
| `--schedule every_1h` / `--schedule "0 */3 * * *"` | the preset or the cron expression; add `--tz <IANA>` to a wall-clock expression and the zone is sent as `timezone` |
| `--time HH:MM [--tz IANA]` | daily: `M H * * *` plus `timezone` (`--tz` defaults to `UTC`) |
| `--weekly mon,wed --time HH:MM [--tz IANA]` | `M H * * 1,3` plus `timezone`; days are `sun`…`sat` or `weekday(s)`, `weekend(s)`, `daily` |
| `--once <ISO>` | `at:` plus the UTC instant. An ISO date-time with an offset or `Z`; a bare local time is read in `--tz`. An unreadable value, an impossible date or a past instant is refused by the CLI before any call (`--once "<value>" is in the past (now is <ISO>).`) |
| `--in 30m` / `2h` / `1d` | `at:` plus now plus the offset (`<N>m`, `<N>h` or `<N>d`) |
| `--tz <IANA>` | the user's timezone; an unknown name is refused (`Invalid --tz "<name>". Use an IANA name (e.g. Asia/Bangkok).`) |

On `update`, giving a cron expression, `--time` or `--weekly` without `--tz` sends `timezone: null`, so a leftover zone never
shifts the new expression; an enum or an `at:` form sends none. `trigger.js explain "<phrase>" [--tz IANA]` prints the
`schedule` (and `timezone` for wall-clock times) it would send, with no API call; it handles phrases such as "every 3
hours", "in 30 minutes", "tomorrow at 8am", "every weekday at 9" and "daily at 9am". `list` shows the stored timezone as a
`(<IANA>)` suffix and a one-time schedule as `once @ <ISO>`.

**Handler flags.** `--next-action run_handler --handler <name>` on `add` or `update`, for every trigger type (the hub
validates; the CLI also refuses a name that fails `^[a-z0-9][a-z0-9-]{0,39}$`). On `add` the two flags must come together
(`--next-action run_handler requires --handler <name>`, `--handler requires --next-action run_handler`); on `update`,
`--handler <other>` alone changes the handler and `--next-action agentic_response` puts the agent back in charge. A handler
cron stays global: the CLI does not default it to the room of the current session. `list` prints, per trigger, the
description, `then: handler <name>` / `agent` / `emit event`, the filter summary and
`last outcome: <lastOutcome> (<lastError>)`.

**Prompt field.** `update --prompt` first lists the triggers and writes `promptTemplate` for an event trigger and `prompt` for
cron and webhook triggers, so the text the dispatcher reads is the one that changes.

### `privos-services` (`service.js`)

`node $SKAWLD_SKILL_DIR/service.js <list|status|logs|add|remove|stop|apply|run>`; see [Room Services](./room-services.md).

| Command | Purpose |
|---|---|
| `add --name screen --kind handler --timeout-seconds 20 --cmd '["sh","AgentFiles/routines/screen.sh"]'` | Declare a handler (`--timeout-seconds` is for handlers only) |
| `run screen --payload-file payload.json` | One dry run with the agent's own payload: exit code, duration, `timedOut`, stdout tail |

Declare and dry-run a handler from a turn in the agent room: the hub looks for it in the agent room's project, and a room
session cannot reach that runtime. The SKILL.md text of both skills tells agents the exit-code protocol (`0` handled, `10`
wake with stdout, anything else a failure), to end every handler with an explicit `exit 0` or `exit 10` and never to `eval`
payload fields; `agent-scheduler` also states the trust boundary ([Agent Routines](./agent-routines.md#trust-boundary)) and
the `NO_REPORT` quiet turn.

## CLAUDE.md Integration

The agent's `CLAUDE.md` includes a Skills section that points to the skill folder:

```markdown
## Skills
You have self-management skills in `.claude/skills/privos-agent-management/`.
Read `SKILL.md` in that folder for trigger management capabilities.
When users ask you to set up automations, schedules, or event listeners, use these scripts.
```

## Rules for Agents

From SKILL.md:

- Maximum 5 triggers per agent
- Always confirm changes with the user
- For webhook triggers, show the generated URL
- When a trigger fires, respond naturally to the prompt
