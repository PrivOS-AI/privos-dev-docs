# Agents Page

One page where a person sees everything their agents do: the agents they own, the agent teams those agents are in, their routines and the bot-to-bot activity of their chains. It exists because each of these lived in a different place (one Agent Settings tab per agent, the team room's member list, an admin-only log), so an owner could not answer "what are my agents doing and what is waiting for me" without visiting all of them.

The page reads what the hub already keeps and adds no state of its own. Every rule (who may add a bot, who may decide an approval, who may stop a chain) stays in the hub; the page shows an action only when the hub would allow it and shows the hub's refusal text when it declines anyway.

## Where it is

| Item | Detail |
|---|---|
| Route | `/agents/:tab?/:id?`, route name `agents`. A bare `/agents` redirects to the first tab; an unknown tab falls back to it |
| Page | `client/views/agents/AgentsPage.tsx` (hub, under `apps/meteor`); the tabs are in `client/views/agents/tabs/` |
| Access | Any signed-in user. What each tab lists is scoped by the hub to the caller, so a user who owns no agent sees an empty state per tab |

### Entry from the sidebar

The sidebar room list has tabs, and the tab bar shows only each tab's icon; the active tab adds its title, and an inactive one keeps its label as a tooltip and aria label. On the **Agents** tab:

- An **Agents overview** row sits under the tab bar and stays fixed while the list scrolls (`AgentsOverviewHeader.tsx`). A plain click opens the page; a modified click (new tab, new window) is left to the browser; on a small phone it also collapses the sidebar, like opening a room.
- With the sidebar collapsed to its rail, the same link is a tile above the rooms, shown only while the Agents tab is the active one.
- The tab lists agent rooms **and the main rooms of the agent teams the user is a member of**, each team with an **Agent team** tag, plus a **paused** tag when the team is switched off. The tab's unread count includes them, and a team row counts the unread of its channels like its own badge does. The list of team rooms comes from `GET agents.a2a.teams.list`; it is read again whenever a team is switched on or off or its roster changes (`useAgentTeamRoomIds`).

## Tabs

| Tab (`:tab`) | Shows | Reads |
|---|---|---|
| **My agents** (`agents`) | One row per agent the user created: name, runtime, super agent flag, number of routines, outcome of the last routine, and buttons to open the agent room and its Agent Settings | `agents.mine`; `agents.triggers.list` once per agent |
| **Teams** (`teams`) | One card per team that any of the user's agents is in: state, the user's agents with their roster status, the roster with owner, runtime, super agent flag and streaming consent, and the actions below | `agents.a2a.teams.overview` |
| **Routines** (`routines`) | One row per trigger of every owned agent: routine, when, then, last run with outcome, an enabled switch, **Run now** and a link to Agent Settings | `agents.mine`; `agents.triggers.list` |
| **Activity** (`activity`) | The bot-to-bot rows of the user's agents and chains, read-only except for approvals and Stop | `agents.a2a.list?mine=true` |

The agents and routines share the Agent Settings card's data (same query key), so a toggle or **Run now** in one place refreshes the other. Reading the lists never wakes a runtime: `agents.handlers.list` is not called here, and a handler's last run still needs **Load handlers** in Agent Settings ([Agent Settings UI](./agent-settings-ui.md#trigger-card)). **Run now** appears under the card's rule: hidden for a room-scoped cron and for an event webhook ([Run now](./agent-settings-ui.md#run-now)).

### Teams: actions and their gate

| Action | Shown when | Call |
|---|---|---|
| **Add to Agent Team** (an own bot that is in the team room but not on the roster, such as a super agent that mirror-joined) | The user holds a team role on the team room, and the team is an agent team that is switched on | `POST agents.a2a.teams.mark { add }` |
| **Remove from Agent Team** (an own rostered bot) | The user holds a team role on the team room | `POST agents.a2a.teams.mark { remove }` |
| Risk tiers editor (Automatic, Approval, Draft only: one action label per line) | A team role, and at least one own rostered bot in the team | `POST agents.a2a.teams.mark { riskTiers }`, which replaces the user's whole entry |
| Agent team switch | The `manage-agent-teams` permission ("Create agent teams"). It is also the only way to turn an ordinary team into an agent team, so it is offered on those too | `POST agents.a2a.teams.setEnabled` |

The **team role** is `callerTeamRole` in the overview: `add-team-member`, else `edit-team-member`, checked on the team room (workspace admins get `add-team-member`); `null` means the user has neither, and no roster action is offered. The hub still applies its own rules on every call (the bot's owner or an admin; the team must be on to add), so the page's check is a convenience, not the authority. Why risk tiers are per owner: one owner's tiers never apply to another owner's bots ([Bot-to-Bot Protocol, Approval flow](./bot-to-bot-protocol.md#approval-flow)). Streaming consent, and the warning when a team room could not be added to a bot's streaming rooms (the 50-room cap), are shown per roster row.

### Activity: reading, approving, stopping

A row click expands it: the message, the reason and the fix (`why`, `fix`), and actions. An **Approve** and **Deny** pair (or one button per option for a `question`) is shown only to the row's approver while the request is open, together with the action and the draft as plain text. A **Pending approvals** switch narrows the list to the requests waiting for the user (`approval=pending`). **Stop** is offered in the expanded row to the chain's human owner; a team moderator or an admin stops a chain by filtering to it and using the filter bar's Stop, the same as the admin log. The table is the one the admin bot-to-bot log uses (`components/a2a/A2aRowsTable.tsx`), without `mine` there.

## Routes

All of them need a signed-in person; a bot caller gets `a2a-not-authorized` on `teams.list` and `teams.overview`, and an empty list from `agents.mine`. Response shapes live in the hub source; this table says who may call and where to look.

| Route | Who | Purpose | Source (under `apps/meteor`) |
|---|---|---|---|
| `GET agents.mine` | the caller, for the agents they created | Active agent bots the caller created, with runtime, agent room id and super agent flag. Built field by field, never `customFields` | `server/services/agents-mine.ts` |
| `GET agents.a2a.teams.list` | the caller, for teams they are a member of | `{ teams: [{ teamId, roomId, enabled }] }`, agent teams only | `server/services/a2a/a2a-teams-overview.ts` |
| `GET agents.a2a.teams.overview` | the caller | Per owned bot, every team it is in, with the roster, `rostered`, the caller's `callerTeamRole` and `callerRiskTiers` | `server/services/a2a/a2a-teams-overview.ts` |
| `POST agents.a2a.approvals.decide` | the row's approver only; an admin or a bot who is not the approver is answered "Only the owner of this agent can decide." | Body `{ rowId, decision: "approved" \| "denied" }` or `{ rowId, option }`. Always HTTP 200 on a valid body: `{ type: "none" }` once decided, `{ type: "error", text }` with the reason otherwise (no longer valid, already decided or expired, kill switch off). Same effects as the card button | `server/services/a2a/a2a-approvals.ts` (`decideByRow`) |
| `GET agents.a2a.list?mine=true&approval=pending` | any person; the role scope still applies | `mine=true` keeps rows whose sender or recipient is a bot the caller created, or where the caller is the chain's human, the sender's human or the approver. For an admin it is the difference between every row and their own. `approval=pending` keeps open requests that have a card | `server/services/a2a/a2a-list.ts` |

`agents.mine` is the only one of these that the Universal Assistant gateway can read (a `user`-tier entry in `lib/universal-bot-rest-read-catalog.ts`); the `agents.a2a.` prefix is excluded from that gateway.

## Related

- [Bot-to-Bot Protocol](./bot-to-bot-protocol.md) — the rules behind the roster, approvals and chains; what an owner sees when a task fails
- [Agent Settings UI](./agent-settings-ui.md) — the per-agent card the Routines tab shares data with
- [Agent Routines](./agent-routines.md) — what a routine is
