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
| **Teams** (`teams`) | One card per team that any of the user's agents is in: state, the user's agents with their roster status, the roster with owner, runtime, super agent flag and streaming consent, and the actions below. Not released: the owner's weekly scorecard, proposals and Evaluation setup on the card ([Agent Team Evaluation](./agent-team-evaluation.md#the-scorecard)) | `agents.a2a.teams.overview`; `agents.a2a.teams.scorecard`, `.proposals.list`, `.mark` |
| **Routines** (`routines`) | One row per trigger of every owned agent: routine, when, then, last run with outcome, an enabled switch, **Run now** and a link to Agent Settings | `agents.mine`; `agents.triggers.list` |
| **Activity** (`activity`) | What needs the user, what is in progress and what is done, then the bot-to-bot rows of the user's agents and chains; read-only except for approvals and Stop | `agents.a2a.list?mine=true` |

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

The tab opens with what needs the owner. Rows carry the sender's [work state](./bot-to-bot-protocol.md#envelope) as
`report` (`state` and `at`; `needs` only when the caller may read the text), and the latest 100 rows of
`agents.a2a.list?mine=true` are sorted into sections above the log
(`client/views/agents/tabs/ActivityTab.tsx`, [hub #755](https://github.com/PrivOS-AI/privos-hub/pull/755)):

| Section | Rows |
|---|---|
| **Needs you** | State `blocked` or `failed`, any row with a `needs`, or an open decision of the user's |
| **In progress** | State `progress` or `waiting`, or a row without a state that is still open |
| **Done** | State `ready` or `dropped`, closed rows, and states a later report from the same bot replaced; the latest 10 |
| **All activity** | The log below, unchanged |

A state counts only as the latest `result` its bot sent on that chain, with or without a state, so a `blocked`
answered by a later `ready` (or by a later result without a state) is done. A stopped chain never shows under Needs you
or In progress. A row shows its state and its need first and expands like a log row. Empty sections are hidden.

A row click expands it: the message, the reason and the fix (`why`, `fix`), and actions. An **Approve** and **Deny** pair (or one button per option for a `question`) is shown only to the row's approver while the request is open, together with the action and the draft as plain text. A **Pending approvals** switch narrows the list to the requests waiting for the user (`approval=pending`). **Stop** is offered in the expanded row to the chain's human owner; a team moderator or an admin stops a chain by filtering to it and using the filter bar's Stop, the same as the admin log. The table is the one the admin bot-to-bot log uses (`components/a2a/A2aRowsTable.tsx`), without `mine` there.

## Knowledge spaces page

Not released: hub [#809](https://github.com/PrivOS-AI/privos-hub/pull/809), branch `feat/agent-knowledge-screens`; the
rules behind it are in [Knowledge Spaces](./knowledge-spaces.md). While `Agent_Knowledge_Spaces_Enabled` is false the
entry button is hidden and both pages only say that knowledge spaces are turned off; the hub refuses the routes anyway.

| Item | Detail |
|---|---|
| Entry | A **Knowledge spaces** button in the Agents page header (`AgentsPage.tsx`), shown only with the switch on; it opens the list |
| Routes | `/knowledge-spaces` (route `knowledge-spaces-index`, the list) and `/knowledge-spaces/:spaceId/:tab?/:itemId?` (route `knowledge-spaces-space`), tabs `notes` (default; an unknown tab falls back to it), `proposals`, `audience`, `settings`. `itemId` is a note's short name on `notes` and a proposal id on `proposals`, so a card's link `/knowledge-spaces/<spaceId>/proposals/<proposalId>` opens that proposal |
| Code | `client/views/knowledge-spaces/` (pages, tabs, create dialog, bulk approval confirm); `client/components/knowledge-spaces/use-knowledge-spaces.ts` holds every call to the `knowledge-spaces.*` routes and `knowledgeErrorText`, which turns a route's `errorType` into a sentence |

- **List.** The spaces the caller administers (`knowledge-spaces.list`): name, kind, a **Restricted** tag, a
  **Disabled** tag for a space that is turned off, and the counts of active notes and of proposals waiting for review.
  **Create space** asks for the kind (Whole workspace only with `manage-knowledge-spaces`), a name of 1 to 80
  characters, the team for a team space, and **Restricted** for owner and workspace spaces only; it lands on the new
  space's notes. The team list holds the agent teams the caller is a member of (`agents.a2a.teams.list`) where the
  caller is an owner or leader of the team room, or all of them for a holder of `manage-agent-teams`; with none it says
  that a team owner or leader can create the space. An agent team the caller is not a member of is not offered, because
  no route lists it.
- **Space page.** A disabled space shows the **Disabled** tag and a line saying that agents do not read it and that its
  notes, proposals and collections are refused (shown as not found) until it is turned on in Settings. Its actions are
  not hidden.
- **Notes.** The order of authority as help text at the top, then title, type, **Approved** or **Not approved** and who
  wrote it. **Bulk approval**: a checkbox on each note no person approved, at most 100 checked. **Approve selected**
  opens a confirm that loads the full text and version of every checked note in one call
  (`knowledge-spaces.notes?slugs=`) and shows each with its title, type and text, rules first, with a line saying how
  many rules become instructions for how many agents. A checked note that is no longer active is named and left out.
  **Approve** stays off until the texts are on screen and sends the version of each text shown, so a note an agent
  changed after that is skipped as stale. The result lists what was approved and each skipped note with its reason. The
  button is off on a disabled space. A note opens with its full text as plain text, its writer, its review date,
  **Edit**, **Retire** (with a confirm) and its **History**. The editor takes a short name (new notes only, optional; a
  short name an active note already uses is refused before saving), a title, a type, a text of up to 16,000 characters
  and an optional **Review by** date for the weekly review, and says how many agents the saved text reaches. An edit
  sends the version it opened, so a change an agent made meanwhile is refused as stale.
- **Proposals.** Pending proposals with operation, type, title, proposer and date, 100 at a time: while more wait, a
  line says how many of how many are shown, with **Load more**. A **Batch** filter lists the batches of the loaded rows;
  with a batch chosen, **Reject this batch** rejects all its waiting proposals after a confirm that gives the batch's
  count. After any decision the lists start again from their first page. A proposal opened by its id (a row click or a
  card's link) is loaded with `knowledge-spaces.proposals?proposalId=`, whatever its page or status. A pending one shows
  the cited message, the proposed text and, for a change or a retire, the current text next to it, with **Approve**,
  **Reject** and **Edit and approve**; the line under it says what approval does, from the edited type while editing.
  **Approve** of a change or a retire is offered only once the current note was fetched after the page opened and is
  still at the version the proposal was made against; otherwise a line says the note changed and to reject or ask the
  agent to propose again. A decided proposal says what became of it (approved, rejected, expired, stale or being
  decided), with no buttons. An id that is missing or of another space shows "There is no such proposal in this space".
  A refusal keeps the buttons; a stale answer keeps the proposal open and shows that the note changed.
- **Audience.** Each entry with its name and level; levels follow the preset and `restricted` (a restricted space offers
  only single agents), `none` is offered for a single agent, and the Agent field suggests agent bots only. Raising to
  Write needs a confirm that says a Write agent also changes and retires at once every note no person approved, other
  agents' and imported notes included. Removing an entry says that the agent then gets the level its group entries give,
  which may be higher or lower (removing None can give access). A refusal is shown under its row, for example the hub's
  message when it refuses to remove the roster entry of a team whose memory moved.
- **Settings.** Name, enabled, restricted (not for a team space), approver (chosen among the space admins), admins
  (owner and workspace spaces), and the **Reviewer agent** of the weekly review, shown only to the approver and listing
  only the approver's own agents. Once the approver is changed in the form, the reviewer select gives way to a line
  saying the change clears the reviewer, and the approver change is sent alone. On a team space whose memory moved, the
  **Enabled** switch is disabled, with a line saying to move the memory back first. **Collect from existing memory**
  starts a harvest: a runner among the caller's agents, the rules files of the caller's agents and, on a team space
  whose memory has not moved, the team memory; it shows the 120,000 character limit, the hub's refusal with the size of
  each source, the batch's status and progress (the source being read, never one past the last), and **Cancel
  collection** while it runs. On a team space, **Move team memory** asks for a confirm that says the roster entry gets
  Write (an agent with its own entry keeps it) and what agents see after the move, then shows the created count and each
  skipped file with its reason (a file whose name is not a short name is to be renamed; `exists` and
  `changed-after-move` compare the file's text with the note). Once moved, the date and **Move back**, whose confirm
  says that a file it overwrites with other text is first copied to the archive folder; it then shows the number of
  notes written and the archive files of team memory that differed from the notes.

## Routes

All of them need a signed-in person; a bot caller gets `a2a-not-authorized` on `teams.list` and `teams.overview`, and an empty list from `agents.mine`. Response shapes live in the hub source; this table says who may call and where to look.

| Route | Who | Purpose | Source (under `apps/meteor`) |
|---|---|---|---|
| `GET agents.mine` | the caller, for the agents they created | Active agent bots the caller created, with runtime, agent room id and super agent flag. Built field by field, never `customFields` | `server/services/agents-mine.ts` |
| `GET agents.a2a.teams.list` | the caller, for teams they are a member of | `{ teams: [{ teamId, roomId, enabled }] }`, agent teams only | `server/services/a2a/a2a-teams-overview.ts` |
| `GET agents.a2a.teams.overview` | the caller | Per owned bot, every team it is in, with the roster, `rostered`, the caller's `callerTeamRole` and `callerRiskTiers` | `server/services/a2a/a2a-teams-overview.ts` |
| `POST agents.a2a.approvals.decide` | the row's approver only; an admin or a bot who is not the approver is answered "Only the owner of this agent can decide." | Body `{ rowId, decision: "approved" \| "denied" }` or `{ rowId, option }`. Always HTTP 200 on a valid body: `{ type: "none" }` once decided, `{ type: "error", text }` with the reason otherwise (no longer valid, already decided or expired, kill switch off). Same effects as the card button | `server/services/a2a/a2a-approvals.ts` (`decideByRow`) |
| `GET agents.a2a.list?mine=true&approval=pending` | any person; the role scope still applies | `mine=true` keeps rows whose sender or recipient is a bot the caller created, or where the caller is the chain's human, the sender's human or the approver. For an admin it is the difference between every row and their own. `approval=pending` keeps open requests that have a card. Each row may carry `report` (the work state) | `server/services/a2a/a2a-list.ts` |

`agents.mine` is the only one of these that the Universal Assistant gateway can read (a `user`-tier entry in `lib/universal-bot-rest-read-catalog.ts`); the `agents.a2a.` prefix is excluded from that gateway.

## Related

- [Bot-to-Bot Protocol](./bot-to-bot-protocol.md) — the rules behind the roster, approvals and chains; what an owner sees when a task fails
- [Agent Settings UI](./agent-settings-ui.md) — the per-agent card the Routines tab shares data with
- [Agent Routines](./agent-routines.md) — what a routine is
