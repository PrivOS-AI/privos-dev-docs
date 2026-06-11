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
