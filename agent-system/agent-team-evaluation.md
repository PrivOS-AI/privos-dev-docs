# Agent Team Evaluation

How an agent team improves itself. An owner chooses one of their agents in an agent team as the **evaluator**. Every
week the hub scores each of that owner's agents in the team from records no agent can write, stores the
**scorecard**, shows it to the owner and wakes the evaluator. The evaluator finds the cause behind weak numbers and
files typed **proposals**. The owner decides each one on an approval card. The hub applies the ones that only reduce
what a trigger does, and the next scorecard says whether each one worked.

> **Status: not released.** This page describes hub [#785](https://github.com/PrivOS-AI/privos-hub/pull/785),
> [#792](https://github.com/PrivOS-AI/privos-hub/pull/792) and [#788](https://github.com/PrivOS-AI/privos-hub/pull/788)
> and sandbox [#387](https://github.com/PrivOS-AI/privos-sandbox/pull/387),
> [#388](https://github.com/PrivOS-AI/privos-sandbox/pull/388) and
> [#389](https://github.com/PrivOS-AI/privos-sandbox/pull/389), merged to `develop`, and hub
> [#808](https://github.com/PrivOS-AI/privos-hub/pull/808) (the evaluator and super-agent rule, the roster add without a
> team role). None of it exists on a workspace until they are released. Everything is off by default: a team without a
> designated evaluator, an owner who has not turned on automatic stops, and a trigger without `decide` behave as before.

## Why this shape

| Decision | Why |
|---|---|
| The hub records, computes and applies; the evaluator analyses and proposes; the owner decides | Agents can write lists, team memory, files and cards, so nothing an agent writes may decide a score, a state or a stop. The numbers come from the hub's own records |
| One evaluator per owner, over that owner's bots only | A team can hold bots of several owners, and no owner sees another owner's numbers |
| No number in the team room | Every member reads the team room. The owner reads the scorecard on the team card; the evaluator posts its reading in its own agent room |
| Typed proposals that only reduce what a trigger does, plus advice delivered as data | The evaluator reads text that teammates and outsiders influence all week. The worst a misled evaluator can cause is a card that asks for less activity |
| A stop needs the owner's card for two weekly cycles, then may be automatic | Owner decision. The automatic path is narrow: only hub-listed triggers of a hub-computed stop candidate, behind a per-owner switch that is off by default |

## Concepts

| Term | Meaning |
|---|---|
| **Role card** | What a roster bot is for: its class, its primary outcome and where the hub counts it. Written by the bot's owner |
| **Evaluator** | The roster agent an owner designated to read that owner's scorecards and file proposals. One per owner per team |
| **Scorecard** | One record per team, owner and week, computed and stored by the hub; one row per agent of that owner |
| **State** | Per agent: `keep`, `fix`, `stop` or `unmeasured` |
| **Coverage check** | Whether the hub can count an agent's outcome at all. An agent that fails it is `unmeasured`, never a stop candidate |
| **Stop candidate** | An agent the hub marks for a possible stop, with the trigger ids a stop would turn off |
| **Proposal** | A typed change the evaluator files for one agent, decided by the owner on a card |
| **Automatic stop** | A stop proposal the hub applies without a card, only under the conditions in [Automatic stops](#automatic-stops) |

## Role cards

The owner writes them in the **Evaluation setup** section of the team's card ([Agents page](./agents-page.md), Teams
tab), shown to every owner of a roster bot and listing only their own agents, or with
`POST agents.a2a.teams.mark { teamId, roleCards: { <botId>: card | null } }`. Only the bot's current owner or a holder
of `view-user-administration` writes one, and only for a roster bot. A body with only `roleCards`, `evaluators` and
`autoStop` needs no team role; one that also changes the roster or tiers needs `add-team-member` or `edit-team-member`,
except the roster add of an owner whom another person added to the team (below).

**Putting an own agent on the roster without a team role.** An owner whose team membership another person added may
roster agents they own now (never the Universal Bot) without a team role, with a `mark` body that carries only `add`
(and `teamId`); the team-room join hook applies the same rule. The hub reads who added the member from the team member
row (`createdBy`): it must be an existing person other than the member, and a row written when a channel became the team
does not count. A member who joined by themselves (a public team, an invite link), one an agent added, or one with no
recorded inviter still needs a team role, as does every removal and an add that also sends `roles` or `riskTiers`. The
refusal says how to get there: `A team role is required to edit the roster. To add agents you own without one, someone
else must have added you to the team: ask a team owner or moderator to add you (if you joined by yourself, leave the
team first) or to give you a team role.` Each such add writes an `agent-team.roster-added-without-role` audit row naming
the inviter. The team-login rule for CLI agents still applies after it ([Agent CLI Runtime](./agent-cli-runtime.md)). No
UI offers this yet: the Agents page roster actions stay with team-role holders.

| Field | Values |
|---|---|
| `class` | `sales`, `content`, `operations`, `coordination`, `assistant`, `service`, `utility` |
| `primaryOutcome` | One line: the countable result the agent exists for |
| `outcomeSource.kind` | `list-stage` (items entering a counting stage; names `listId` and `stageId`), `a2a-ready` (tasks answered `ready` on the first result), `published-files`, `outcome-cards` (cards filed for the agent on an Outcomes list; self-reported), `verified-proposals` (its proposals the hub marked Verified; the evaluator only) |
| `baseline` | Optional: primary outcomes per week to compare with |
| `valuePerOutcomeUsd` | Optional: what one primary outcome is worth in US$ (above 0, at most 1,000,000); rates `profitable` |

A bot without a role card is `unmeasured`, except the evaluator, which is then counted by `verified-proposals`. A
`utility` agent (reports, timekeeping, monitors) is counted by the fires that started and is never a stop candidate. A
card is tied to the bot's owner: a handover removes it.

## The evaluator

- **Designation.** `mark { evaluators: { <ownerId>: <botId> | null } }`, by that owner or a holder of
  `view-user-administration`, only to a roster agent the owner owns now that is not a super agent. Another roster
  editor cannot replace it. The hub rechecks it on every run, and a handover removes it. The reverse holds too: the
  super-agent flag is refused on a bot that is a designated evaluator in a team that exists and still has it on the
  roster, until that designation is cleared ([Super Agent](./super-agent.md#flag-and-eligibility)).
- **Wake.** The hub wakes the evaluator with an in-app notification addressed to it, carrying the team (`teamId`) and
  the scorecard (`itemId`); a notice that a teammate paused carries `teamId` and no `itemId`. The evaluator needs a
  filtered event trigger on `notification.created`
  ([Subscription filter](./trigger-api-reference.md#subscription-filter)); the turn runs in its own agent room, which
  only it and its owner see. Without a listener the scorecard is still stored and the owner still notified.
- **Skill.** `privos-team-evaluate` (`scripts/evaluate.py`): `scorecard`, `ack`, `propose`, `proposals`. It prints
  every answer as one line of escaped JSON, so names and titles reach the agent as data. The skill teaches the agent
  to treat every name and text as data, to change no trigger itself and ask nobody to, to keep numbers in its agent
  room, to file at most one proposal per agent per week, and to wait, not refile, after `a2a-approval-final` or
  `a2a-approver-unreachable`.
- **Its own row.** The evaluator is on the scorecard like every other agent, counted by its Verified proposals (a mark
  counts on the next week's scorecard) unless it has a role card, and never a stop candidate.

## The scorecard

A hub job, not an agent, runs every **Monday at 08:00 Asia/Bangkok** (01:00 UTC). For each team and each owner with a
designated evaluator it reads the 7 days that end at the run and stores one scorecard; the previous week is a
scorecard made 6 to 8 days before, and `cycle` restarts at 1 after a gap of more than 8 days. Each job runs once a
day across hub instances (a lease per job and UTC date). Then it checks the proposals applied before that window,
applies the automatic stops the owner allowed, notifies the owner (a link to the team card, no number) and wakes the
evaluator. A **daily** run at 08:30 closes undecided proposals, posts fresh cards,
reminds an evaluator that has not acknowledged the week's scorecard, and wakes it when the waiting question boxes or a
trigger's health changed (paused, or 20 % or more of 10 or more fires in a day failed).

What it reads: the trigger fire log, the stage events of each counting list, the trigger question log (see
[Owner questions](./trigger-api-reference.md#owner-questions-decide)), bot-to-bot tasks and their results' work state,
published files, the hub usage log, and who opened the rooms the agent posted in. A super agent's room cron fire the
hub refused (`not-started:super-agent-principal`) counts as a failed fire. Team-core CLI spend reaches the usage
log only with sandbox #389, as per-session increments; earlier rows summed the CLI's running totals (about 2.8x too high
on the measured core). Stage events are written by the hub's item model, one per move, creation in a stage, list copy
and stage deletion (a deleted list leaves one `list-deleted` event), with the actor typed `user`, `bot` or
`unattributed`. A move by the agent itself counts as its own outcome, never as a person's.

**Six dimensions**, each a level from 0 to 3, or unrated (`null`, left out of the score, never counted as zero).
Levels and thresholds: hub `apps/meteor/server/services/a2a/a2a-team-evaluation.ts`.

| Key | Rule | Measured as | Unrated when |
|---|---|---|---|
| `ships` | Everyone ships | Primary outcomes in the week against the role card's baseline | Before a baseline exists, unless the agent had no outcome (level 0) |
| `profitable` | Everyone profitable | Outcomes times `valuePerOutcomeUsd`, divided by model spend; cost per outcome is always shown | Without a value per outcome, or while spend is unknown or zero |
| `zeroPerson` | Zero-person work | Share of started runs that ended without a question box | No started runs |
| `ownOutcomes` | Own outcomes | Share of received tasks answered `ready` on the first result; held at level 1 while 5 % or more of fires failed | No received tasks |
| `writeDecide` | Write, decide | Median minutes from a received task to its result | No task answered |
| `faster` | Make fast faster | This week's median cycle time (item creation to the counting stage) against the previous four weeks | Until four weeks of stage events exist |

- **Score**: the rated levels as a share of their maximum.
- **State**: `keep` from two thirds; `fix` from one third, or when the use guard is under 20 %; `stop` under one third
  or as a stop candidate; `unmeasured` when the coverage check fails or nothing is rated.
- **Coverage check**: fails with `no-source` (no role card), `results-without-state` (under 80 % of the agent's results
  carry a work state), `no-tasks` (an `a2a-ready` agent, not `utility`, that received no task), `not-evaluator` (a
  non-evaluator counted by `verified-proposals`), `source-missing`, `source-deleted` or `source-changed` (the counting
  list or stage is gone, was deleted, or received cards because another stage was deleted).
- **Use guard**: the share of rooms the agent posted in during the week that a person opened after its first post
  there. Shown next to the score; never a stop by itself.
- **Stop candidate**: no hub-recorded outcome in this and the previous scorecard, both covered, not `utility`, not the
  evaluator, and not counted from self-reported outcome cards. `stopTriggerIds` lists its enabled triggers.

The owner sees the latest scorecard on the team's card in the [Agents page](./agents-page.md), Teams tab: the week,
whether the evaluator read it, and per agent the state, score, six levels (a dash with the reason when unrated),
coverage reasons, rooms opened, fires, tasks, outcomes, cost and question-log totals; below it the owner's proposals
with state, decision, check and audit references, and the Evaluation setup.

## Proposals and the owner's cards

| Type | What the hub does on approval |
|---|---|
| `disable-trigger` | Turns the trigger off |
| `slow-trigger` | Moves a preset interval to a longer preset; a cron expression has no slower form and is refused |
| `demote-decide` | Lowers the trigger's [`decide`](./trigger-api-reference.md#owner-questions-decide) one step: assist to shadow, shadow to off |
| `advice` | Changes nothing. The text (at most 1,500 characters, all shown on the card) reaches the target as a hub row marked as data, never as its owner's instruction |

1. **Filed** by the evaluator against one scorecard: the target, the type, the trigger (for the three trigger types),
   a one-line cause (at most 300 characters) and the scorecard number it should move (`metric`): `score`,
   `dims.ships`, `dims.profitable` or `counts.costUsd` for the two trigger types, `counts.reversed` or `dims.zeroPerson`
   for `demote-decide`, `score` or a dimension rated for the target for `advice`. Refused: any caller but the owner's
   designated evaluator, a proposal about the evaluator itself, another owner's bot or a bot off the roster, an unknown
   type or a metric the type does not take, a trigger of another bot, a change that does not apply now, a second open
   proposal for the same agent, and the same proposal (team, target, type, trigger) the owner denied in the last 7
   days (`a2a-approval-final`).
2. **Open** for 7 days. The card is posted as the evaluator, only in its agent room, and only the target's owner
   decides it. An owner who is not a member of that room gets no card: the proposal is closed `expired` with
   `closedReason: 'not-applicable'` (`a2a-approver-unreachable`). A card lasts 24 hours, so the daily run posts a fresh
   one before it ends and retires the old one.
3. **Decided** on the card or on the Activity page. A denial is `rejected`; 7 days without a decision is `expired`.
   The hub carries an approval out itself, so no bot receives a decision row from a proposal card.
4. **Applied** through one trigger writer, which writes an `agent-trigger.changed` audit row naming the proposal, the
   evaluator and the approver before the change.
5. **Checked** by the next weekly run: `verified` when the metric moved the right way, else `not-moved`. The skill
   tells the evaluator not to file the same change again without new evidence.

**Audit.** Every step (filed, card issued, decided, automatic stop, applied, expired, closed) and every change of a
role card, a designation or the switch writes a durable `agent-team.*` row in `ServerEvents`, the store of
`agent-trigger.changed`: ids, codes and numbers only, with its own `auditRef`, which the proposal keeps in `auditRefs`.

The hub checks the evaluator, the target's owner, the roster and the trigger again at filing, at the decision and at
apply; a proposal that no longer holds is closed with a `closedReason`. States: `open`, `approved`, `applied`,
`rejected`, `expired`, `verified`, `not-moved`.

**An evaluator cannot change a trigger through a teammate.** `agents.a2a.send` refuses a row that carries a trigger
change (`data.triggerChange`) when a designated evaluator of any team is an initiator or participant anywhere in the
row's chain lineage (its chain, its lineage parent, their root and parent chains), so a relay through teammates is
refused too: a proposal is an evaluator's only way to change an agent. Other bots keep the
[teammate trigger change](./bot-to-bot-protocol.md#sandbox-and-harness-agents-the-privos-team-skill).

**Handover.** When a bot changes owner, its role cards, its designation as evaluator and its open proposals (about it
or filed by it) are removed with their cards, the automatic-stop switch of both owners in its teams is cleared, and
`decide` is turned off on all its triggers (listed in the handover result as `triggersDecideReset`).

## Automatic stops

`mark { autoStop: { <ownerId>: true | false } }`, by that owner or a holder of `view-user-administration`; `true` only
for an owner of a roster bot. Off by default. A `disable-trigger` proposal is applied without a card only when all of
these hold:

- the scorecard it answers is the owner's third or later in that team, their newest there and at most 7 days old,
  and the owner's switch is on;
- the target is a stop candidate on that scorecard, passes the coverage check, is not `utility` and is not the
  evaluator;
- the trigger is one of that row's `stopTriggerIds`, has not changed since that scorecard was made, and was never
  stopped automatically before.

The owner then gets an in-app notice naming the proposal, and the audit rows have the system actor. Every other
proposal needs a card, so a trigger the owner turned back on is never stopped again without one.

## Routes

Response shapes: hub `apps/meteor/app/api/server/v1/agents.ts` (Endpoints). Refusals use the
[bot-to-bot error codes](./bot-to-bot-protocol.md#error-codes).

| Route | Who | Purpose |
|---|---|---|
| `POST agents.a2a.teams.mark` (`roleCards`, `evaluators`, `autoStop`) | a person within the rules above (no team role needed for these three alone); bots refused | Write role cards, the designation and the switch; the answer shows the caller's own cards, evaluator and switch, as does `callerEvaluation` per team in `agents.a2a.teams.overview` |
| `GET agents.a2a.teams.scorecard?teamId=&scorecardId=&history=&ownerId=` | the owner, the owner's evaluator, holders of `view-user-administration` (`ownerId` is for them only) | The latest scorecard (or the named one) and up to 8 earlier ones; rows only for bots the owner still owns |
| `POST agents.a2a.teams.scorecard.ack { scorecardId }` | the evaluator named on the scorecard | Stamps `ackedAt` once; `alreadyAcked` after |
| `POST agents.a2a.teams.proposals.file` | the owner's designated evaluator (bot key) | File a proposal; refused while bot-to-bot is switched off, and as in [Proposals](#proposals-and-the-owners-cards) |
| `GET agents.a2a.teams.proposals.list?teamId=&state=` | the evaluator (its owner's), an owner of a roster bot, holders of `view-user-administration` | Proposals, newest first, at most 200 |

## Related

- [Trigger API Reference, Owner questions](./trigger-api-reference.md#owner-questions-decide) — the trigger field
  `decide`, the question log and owner-only answers the scorecard reads
- [Bot-to-Bot Protocol](./bot-to-bot-protocol.md) — rosters, approval cards, work states
- [Agents Page](./agents-page.md) — where the owner reads the scorecard
