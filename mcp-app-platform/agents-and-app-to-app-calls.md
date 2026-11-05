# Agents and App-to-App Calls

How room agents and other apps call your app's tools, what you declare to allow it, and what you must check
inside a tool. See [Manifest Format](./developer-guide.md#5-manifest-format) and
[Tool Execution](./apis/rest-tool-call.md).

## Who can call your tools

Every call goes through `POST /api/v1/mcp-apps.tool-call`. The Hub classifies the caller by its credential, not by
its user: an installation bot is one user with two credentials.

| Caller | How it is recognized | Tools | Rooms |
|--------|----------------------|-------|-------|
| **Person** | A human user | As the app's UI allows | Rooms they can access |
| **Your own server** | Your installation bot on a login token or DPoP, no session header | All your tools | Unchanged |
| **Your own LLM agent** | Your installation bot on its bearer key | All your tools | Rooms it can access |
| **Another app's server** | That app's installation bot on a login token or DPoP, no session header, installation active | `shared` tools, under `enforce` (see [Enforcement modes](#enforcement-modes)) | Any room where your app is installed, except DMs and inbox rooms. Membership is not required |
| **Any other agent** | Room agent, guest agent, chat bot, another app's LLM agent | Real tools that are not UI entry points. Under `enforce`, `shared` tools only | Only the room of its session, and it must be a member |

A bot never runs another app's built-in `mcpapp.*` tools, and a bot calling another app must send `roomId`.

## Declaring shared tools

List the tools others may call under the top-level `toolAccess` of the manifest:

```json
{
  "tools": [
    { "name": "mcpapp.crm.dashboard" },
    {
      "name": "mcpapp.crm.upsertContact",
      "title": "Create or update a contact",
      "description": "Creates the contact, or updates it when the email already exists.",
      "inputSchema": { "type": "object" }
    }
  ],
  "toolAccess": { "mcpapp.crm.upsertContact": "shared" },
  "capabilities": { "verifiedActor": true, "caller_app": true, "cross_room": true }
}
```

- Every key must be a name in `tools[]` (200 at most). The only value is `"shared"`.
- `tools[]` must match what your server returns from `tools/list`, or a v3 install fails its readiness check.
- `"toolAccess": {}` means **nothing is shared**, and that stays true across upgrades.
- No `toolAccess` at all means **not declared**: shared-tool rules are not applied to your app.
- A UI entry point, such as the room tab tool, can never be called by an agent, even when `shared`.
- Share only what is needed. Mark read tools as read-only in `description`, and make write tools idempotent.
- Agents read `description` and `inputSchema`, so write them for a reader who has no other context.

An upgrade that shares a tool the installed version does not share needs the workspace owner's consent, like a
widened permission (`TOOL_ACCESS_WIDENED`). The consent page and the Hub's update notice list the shared-tool
changes. Removing shared tools needs no consent.

## Checking the caller in a tool

The Hub signs the caller's details and `@privos_ai/app-server` (0.13.2 or later) puts them on `context.actor`:

| Field | Meaning |
|-------|---------|
| `actor.roomId` | The room of the call. For another app's call the caller chooses it: it proves only that your app is installed there |
| `actor.callerApp` | The calling app's id, when another app's server made the call |
| `actor.crossRoom` | `true` when another app's server made the call and its bot cannot access the room (not a member of a private room, or no room named). Absent when that bot can access it |
| `actor.principalType` | `'bot'` when the caller is automation. Absent means "not marked", not "human" |

What you receive depends on the path:

| | User token (Relay, Direct) | App Cluster dispatch |
|---|---|---|
| Actor | Always | Only with `capabilities.verifiedActor` |
| `callerApp` | Always, for another app's call | Only with `capabilities.caller_app` as well |
| `crossRoom` | Always, for a call from outside the room | Only with `capabilities.cross_room` as well |
| `principalType` | Set for bots | Never |

Deny by default:

- Never authorize a bot on `actor.roomId` alone.
- Accept `actor.callerApp` only for apps you trust. For a call with `crossRoom`, also check your own configuration,
  for example the rooms an admin allowed that app to write to.
- Never take the target room from a tool argument when the caller is an agent or another app.

```ts
const actor = assertActorAvailable(context, 'enforce');
if (actor.callerApp && !myAllowedCallers.has(actor.callerApp)) {
  throw new Error('Caller app not allowed');
}
if (actor.callerApp && actor.crossRoom && !allowedOutsideRoom(actor.callerApp, actor.roomId)) {
  throw new Error('Caller app not allowed in this room');
}
if (actor.principalType === 'bot' && isWrite && !myBotWriteAllowList.has(actor.userId)) {
  throw new Error('Automation callers may not write');
}
```

## How agents reach your app

Every sandbox agent has the default skill `privos-mcp-apps`, with the commands `list`, `tools`, `describe` and
`call`. It works on the apps of the current room, so an agent can use your tools with no app-specific skill. Ship a
skill of your own only to teach the agent how your app should be used; build it on `privos-mcp-apps`.

An agent runs at most 4 tool calls at once.

## Enforcement modes

Two workspace settings control these rules.

`MCP_Apps_Agent_Guard_Mode`, default `enforce`, covers the agent rules, the bot rules for built-in tools and
`roomId`, the room access check and the `rest-call` rule. `off` is a manual kill switch for all of them.

`MCP_Apps_Shared_Tools_Mode`, default `off`, covers the shared-tool rules:

| Mode | Effect |
|------|--------|
| `off` | No shared-tool check and no audit. Another app's server can call any tool your app declares |
| `audit` | Calls that would be refused are recorded. Nothing is blocked |
| `enforce` | Refuses them. `TOOL_NOT_SHARED`, the DM/inbox rule and a missing room apply only to a target app that has `toolAccess`. The two admin switches below always apply |

Until `enforce` is on, authorize inside your tool as described above. That check is what protects you in every mode.

## Admin switches

Workspace admins control two switches:

- **Agents and other apps may call this app**, per room, on by default, in the room's app settings. Off blocks agents
  in that room, and other apps' servers under `enforce`.
- **This app may call other apps**, per calling app, on by default, in the app's admin settings. Off blocks that app's
  server under `enforce`.

## Refusals

| HTTP | Code | Meaning |
|------|------|---------|
| 403 | `ROOM_ID_REQUIRED` | A bot called another app without `roomId` |
| 403 | `ROOM_ACCESS_DENIED` | The caller cannot access the room |
| 403 | `ROOM_BINDING_MISMATCH` | An agent named a room other than its session room |
| 403 | `ROOM_BINDING_REQUIRED` | The app is not installed in that room |
| 403 | `ROOM_TYPE_NOT_ALLOWED` | Another app's server targeted a DM or inbox room |
| 403 | `TOOL_NOT_AGENT_CALLABLE` | A UI entry point, a built-in tool of another app, or a tool not in `tools[]` |
| 403 | `TOOL_NOT_SHARED` | The tool is not `shared` |
| 403 | `AGENT_TOOL_CALL_DISABLED` | The room's switch is off |
| 403 | `APP_TO_APP_DISABLED` | The calling app's switch is off |
| 403 | `SESSION_HEADER_NOT_ALLOWED` | A session-room header on anything but an agent's bearer key |
| 403 | `APP_IDENTITY_MISMATCH` | A bot named another app on `mcp-apps.rest-call` |
| 429 | | The agent already has 4 calls in flight |
| 503 | `APP_OFFLINE` | A Relay app is offline (agent calls) |
| 504 | `TOOL_CALL_TIMEOUT`, `APP_DISCONNECTED` | The app did not answer (agent calls) |
