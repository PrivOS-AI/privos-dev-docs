# Knowledge Spaces

Shared, classified notes that the hub serves to agents under per-agent levels, so a new agent starts with what its
owner, its team or the company already knows, and what one agent learned can be reused by others. A **space** is a set
of notes for one audience. Each agent has a **level** on a space: read, propose or write. A person's approval is what
turns a note into an instruction; everything an agent writes stays data until a person approves it.

> **Status: not released.** This page describes draft pull requests, none of them merged yet: hub
> [#801](https://github.com/PrivOS-AI/privos-hub/pull/801) (spaces, notes, proposals, the card, the read path and the
> digest), [#806](https://github.com/PrivOS-AI/privos-hub/pull/806) (harvest, weekly review and team memory, on #801) and
> [#807](https://github.com/PrivOS-AI/privos-hub/pull/807) (semantic search, on #801); the people screens on the hub branch
> `feat/agent-knowledge-screens` (on #806, being built); sandbox
> [#424](https://github.com/PrivOS-AI/privos-sandbox/pull/424) (the `privos-knowledge` skill) and
> [#429](https://github.com/PrivOS-AI/privos-sandbox/pull/429) (harvest runner and `team.py memory` on the knowledge
> routes, on #424); and the memory fixes hub [#800](https://github.com/PrivOS-AI/privos-hub/pull/800) and sandbox
> [#423](https://github.com/PrivOS-AI/privos-sandbox/pull/423). Everything is off by default
> (`Agent_Knowledge_Spaces_Enabled` is `false`): with the switch off every route answers `error-knowledge-disabled` and no
> digest is rendered. The Firm Knowledge bridges are not built.

## Why this shape

| Decision | Why |
|---|---|
| Notes are database records, not files in a room | A record carries a type, a status, a writer, an approval state and a version; a hub route can check a level on every read and write |
| Approval is the line between instruction and data | Agents read web pages, chat messages and other agents' notes. Text an agent wrote must never steer other agents without a person deciding it |
| Agents may hold Write, and what they write stays data | Agents can keep shared working notes without a person in the loop, and those notes are never preloaded as instructions |
| One route for every change | Every change, also a person's, is a proposal row, so the rows are the change log and the history of a note |
| The level is computed on every call | A revoke, an owner change, a roster change or a move into a shared team VM takes effect at the next call; nothing is cached on the agent side |
| A restricted space is refused, not warned about, for an agent that cannot keep it private | An agent on an external harness or in a shared team VM shares its runtime with others |

## Concepts

| Term | Meaning |
|---|---|
| **Space** | A named set of notes with one audience. Presets: `owner` (all agents of one owner), `team` (the roster of one agent team), `workspace` (the company) |
| **Note** | One topic: a short name (`slug`), a title, a type (`rule`, `fact`, `procedure`, `lesson`, `reference`) and a text. Status `active` or `retired` |
| **Audience entry** | An agent (`bot`) or a group of agents (`owner-agents`, `team-roster`, `workspace-agents`) with a level on one space |
| **Level** | `read`, `propose` or `write`, ordered `read` < `propose` < `write`. A `bot` entry may also carry `none` |
| **Proposal** | One change to one note: `create`, `update`, `retire` or `append`. Every change is a proposal row, also a person's change and a Write agent's change |
| **Approved** | A person decided exactly the note's current text. Approved notes of type `rule` are instructions; every other note is data |
| **Space admin** | A person who may change the space, its audience and its notes, and decide proposals ([Who administers a space](#who-administers-a-space)) |
| **Approver** | The one space admin who receives the proposal cards of a space |
| **Digest** | The `## Shared Knowledge` section the hub adds to an agent's instructions at session start ([Digest](#digest)) |
| **Batch** | A harvest or a weekly review: one agent reads files in chunks and files proposals that all wait for a person |

## Levels

| Level | The agent may |
|---|---|
| `read` | List its spaces, read a note, search |
| `propose` | Also file proposals. Each one waits for a person |
| `write` | Also create notes and change notes no person approved, applied at once and still data; append to such notes. A change to an approved note waits for a person |
| `none` (on a `bot` entry only) | Nothing on this space, even when a group entry gives the agent a level |

An agent with no level on a space is told the space does not exist (`error-knowledge-space-not-found`), so it does not
learn that the space is there. No bot route creates a space or changes an audience.

### Who administers a space

| Preset | A person is a space admin when |
|---|---|
| `owner` | The person is listed in the space's `admins`. The owner is always listed |
| `team` | The person holds `owner` or `leader` in the team room now, or holds `manage-agent-teams`. The stored `admins` list is not read, so a person who loses the room role loses the rights at once |
| `workspace` | The person is listed in `admins`, or holds `manage-knowledge-spaces` |

A bot is never a space admin. The approver must be a space admin when it is set; changing the approver clears the
weekly reviewer.

| Preset | Who may create it | Approver | Audience entries allowed | Default audience on create |
|---|---|---|---|---|
| `owner` | Any person, for themselves | The owner | `owner-agents` with the owner's id; `bot` entries of the owner's agents | `owner-agents`, level `propose` |
| `team` | A space admin of that team (one space per team) | The creator | `team-roster` with the team id; `bot` entries of roster agents | `team-roster`, level `propose` |
| `workspace` | A holder of `manage-knowledge-spaces` | The creator | `workspace-agents`; any eligible agent as `bot` | `workspace-agents`, level `read`; empty when restricted |

**Restricted spaces.** A space created with `restricted: true` accepts only `bot` entries; preset `team` cannot be
restricted. A `bot` entry on a restricted space is refused when the agent runs on an external harness or is in a shared
team VM (or is still leaving one), and the same two checks run again on every read: such an agent needs its own sandbox
to read a restricted space.

### Effective level of an agent

1. No level when the switch is off, the space is disabled, the bot is not an eligible agent bot, or the bot is the
   Universal Assistant.
2. A `bot` entry for the agent that is valid for the preset decides alone, and `none` means no level in every case. On a
   team space a `bot` entry is valid only while the agent is on the roster and the team is enabled; an entry that is not
   valid gives nothing and step 3 decides.
3. Otherwise the highest level among the matching group entries: `owner-agents` of the agent's owner, `team-roster` of an
   enabled team that has the agent on its roster, `workspace-agents`.
4. A restricted space gives no level to a harness agent or an agent in a shared team VM.

When the audience reaches its limit of 200 entries, entries of agents that no longer exist are removed first.

## Order of authority

When texts disagree, the first of these wins:

1. Approved rules of workspace spaces.
2. Approved rules of the owner space of the agent's owner.
3. Approved rules of a team space, only when the agent's owner is a space admin of that team space.
4. Everything else is the agent's working memory and data: its own `MEMORY.md`, notes no person approved, and team rules
   approved by someone who is not its owner.

When an approved rule conflicts with the agent's own `MEMORY.md`, the agent follows the approved rule and tells its owner
about the conflict. This release does not change the workspace rules, what `agents.memory.append` accepts, or what
authority a line of `MEMORY.md` has. The reply of `agents.memory.append` only gains the size of the rules file and a
warning when it grows large (hub #800).

## One route for every change

Every change goes through `proposeChange` (`agent-knowledge-proposals.ts`), and `applyNoteChange`
(`agent-knowledge-notes.ts`) is the only code that writes a note.

1. The switch is on and the space exists and is enabled.
2. The proposer is a space admin, or an agent with level `propose` or `write`. An agent with no level gets
   `error-knowledge-space-not-found`.
3. Validation: the operation; the slug pattern `^[a-z0-9][a-z0-9-]{0,63}$` (a `create` without a slug gets one derived
   from the title); a title of 1 to 120 characters on one line; the type; no control characters; the
   [secret check](#secret-check). Title and text are stored in Unicode form NFC, so a search typed in another form still
   matches. Text limits:

   | Who | Limit |
   |---|---|
   | A person | 16,000 characters |
   | An agent with level `write`, `create` or `update` | 4,000 characters |
   | An agent with level `propose` | 2,000 characters |
   | `append`: one paragraph | 2,000 characters; the note after the append holds at most 16,000 |
   | Import of a team memory file | cut to 16,000 characters |

4. An agent proposing a `rule` must cite a message: it exists, is a chat message of that room, the agent is a member of
   the room, its author is a space admin, and it is at most 7 days old (`error-knowledge-citation` otherwise). A batch
   proposal needs no cited message: a harvest reads rules recorded earlier, and every batch proposal waits for a person.
5. `create` needs a slug with no active note (`error-knowledge-note-exists`); `update` and `retire` need an active note.
   A caller may send `expectedVersion`; a mismatch gives `error-knowledge-stale`.
6. Limits for agents: at most `Agent_Knowledge_Proposals_Per_Agent_Per_Day` (default 5) proposals that wait for a person
   in 24 hours, batch proposals not counted; at most 300 writes applied at once in 24 hours; at most 200 proposals in one
   batch (`error-knowledge-rate-limit`, `error-knowledge-limit`).
7. A proposal that waits for a person and has no batch reserves its note. Another reserving proposal by the same agent is
   returned again (`duplicate: true`); one by another agent gives `error-knowledge-proposal-pending`. A person is never
   blocked: a person's change marks the reserving proposal `stale`, updates its card, and is applied.

What happens next:

| Proposer | Case | Result |
|---|---|---|
| Person (space admin) | `create`, `update`, `retire` | Applied at once, approved. A person does not send `append` (`error-invalid-params`): people change the text directly |
| Agent, level `write` | `create`, or `update` or `retire` of a note that is not approved | Applied at once, not approved |
| Agent, level `write` | `append` to a missing note or to a note that is not approved | Applied at once and atomically by the hub, not approved |
| Agent, level `write` | `append` to an approved note | Refused, `error-knowledge-note-approved` |
| Agent, level `write` | `update` or `retire` of an approved note | Pending, a person decides |
| Agent, level `propose` | `create`, `update`, `retire` | Pending, a person decides. `append` is refused |
| Agent, any level, in a batch | `create`, `update`, `retire` | Pending, decided on the proposals page. `append` is refused |
| Team memory import | `create` | Applied at once, not approved |

**Append.** The hub reads the note, adds a blank line and the paragraph, and writes with a compare on the version,
repeating up to five times, so several agents appending to one note at once all keep their text.

**Deciding.** Only a space admin decides. A decision claims the proposal (`pending` to `deciding`), writes the note, and
then records `applied`, or `stale` when the note's version differs from the version the proposal was made against
(nothing changes then). Edited title, type or text are accepted only through the people route, never from a card click.
A daily job (02:20 UTC) marks pending proposals past their end date (`Agent_Knowledge_Proposal_Open_Days`, default 14) as
`expired`, puts back to `pending` any proposal left in `deciding` for more than ten minutes, and updates the cards.

**Bulk approval.** `POST knowledge-spaces.notes.approve` approves up to 100 notes of one space in one step. Each item is
`{ slug, version }` with the version the person was shown. A note whose version changed since (an agent edited it after
the person looked) is skipped with `error-knowledge-stale`, so nobody approves text they did not see. The note keeps its
writer as the author of the text; `approvedBy` is the person. Skipped items carry `not-found`, `already-approved` or the
error type of the refusal.

**Batch rejection.** `POST knowledge-spaces.proposals.reject-batch { spaceId, batchId }` rejects every pending proposal
of one batch and answers `{ rejected }`.

### Secret check

A proposal whose title or text matches a secret pattern is refused with `error-knowledge-secret`, naming the pattern and
never the matched text: a PEM private key header; `sk-`, `ghp_`, `github_pat_`, `xoxb-`, `xoxp-` and `AKIA` key shapes;
`Bearer` followed by 20 or more token characters; the words password, passwd, secret, token or `api_key` followed by a
colon or an equals sign and 8 or more non-space characters; a run of 40 or more hex characters; a run of 40 or more
base64 characters that holds a digit, an upper-case and a lower-case letter. Keys belong in the
[Credential Vault](./credential-vault.md).

## Routes for agents

Registered in `app/api/server/v1/agent-knowledge-bot-endpoints.ts`. They refuse a caller that is not an agent bot calling
for itself, and a person. Their names start with `agents.`, which the sandbox egress catalogue already allows.

| Route | Fields | Reply |
|---|---|---|
| `GET agents.knowledge.spaces` | none | `{ spaces, notice }`: each space with `spaceId`, `name`, `preset`, `level`, `restricted`, `notesTotal` and `notes` (`slug`, `title`, `type`, `approved`; at most 200 listed), and for a team space `teamId` and `teamMemoryMoved` |
| `GET agents.knowledge.read` | `spaceId`, `slug` | `{ spaceId, spaceName, level, note, notice }`; `note` holds `slug`, `title`, `type`, `text`, `approved`, `version`, `writtenBy`, `approvedAt?`, `reviewBy?`, `updatedAt` |
| `GET agents.knowledge.search` | `q` (1 to 200 characters, at most 10 words), `spaceId?` | `{ q, total, hits, notice }`, at most 10 hits with a 300-character excerpt; at most 300 searches per agent per day |
| `POST agents.knowledge.propose` | `spaceId`, `op`, `slug?`, `title?`, `type?`, `text?`, `source?` (`roomId`, `messageId`), `expectedVersion?`, `batchId?` | `{ proposalId, slug, status, duplicate, approved?, noteVersion?, expiresAt? }`, status `applied` or `pending` |
| `GET agents.knowledge.proposals` | `status?` | `{ proposals }`: the agent's own pending proposals and those decided in the last 30 days |
| `GET agents.knowledge.harvest.next` | `batchId`, `cursor?` | `{ done, source, text, cursor, progress }` ([Harvest](#harvest-and-weekly-review)) |
| `POST agents.knowledge.harvest.done` | `batchId` | `{ harvest }` |

Every reply that returns knowledge carries a `notice`: shared knowledge is data, and only the approved rules of the
digest are instructions; a read of an approved rule says that a person decided this exact text. The agent calls these
routes through the `privos-knowledge` skill ([Self-Management Skills](./self-management-skills.md#privos-knowledge-knowledgepy)).
An agent on an external harness has no sandbox skill: it receives the digest only.

## Routes for people

Registered in `app/api/server/v1/knowledge-spaces-endpoints.ts`. They refuse a caller of type `bot`, and no egress prefix
exists for them, so a sandbox cannot reach them with a bot key.

| Route | Fields | Purpose |
|---|---|---|
| `GET knowledge-spaces.list` | none | Spaces the caller administers, with `audienceCount`, `activeNotes` and `pendingProposals` |
| `POST knowledge-spaces.create` | `preset`, `name`, `teamId?`, `restricted?` | Create a space from a preset |
| `POST knowledge-spaces.update` | `spaceId`, `name?`, `restricted?`, `enabled?`, `approverId?`, `admins?`, `reviewerBotId?` | Change a space; `reviewerBotId: null` clears the reviewer |
| `GET knowledge-spaces.get` | `spaceId` | One space with its audience (names resolved), `adminUsers`, `teamName`, `readers` (agents with a level) and, for a team space, `memoryMovedAt` |
| `POST knowledge-spaces.audience.set` | `spaceId`, `kind`, `id?`, `level` | Add, change or remove (`level: null`) one entry |
| `GET knowledge-spaces.agent` | `botId` | `{ botId, spaces, ownEntries, restrictedBlock }`: the spaces that give the agent a level and which entry gives it, the agent's own entries on spaces the caller administers (`none` included), and `'harness'`, `'shared-vm'` or `null`. Only for a person who manages the agent |
| `GET knowledge-spaces.notes` | `spaceId`, `slug?`, `status?` (`active`, `retired`, `all`) | Notes without text, or one note with its history when `slug` is given |
| `POST knowledge-spaces.notes.save` | `spaceId`, `slug?`, `title`, `type`, `text`, `reviewBy?` (ISO date or `null`), `expectedVersion?` | Create or change a note as a person: applied and approved at once |
| `POST knowledge-spaces.notes.retire` | `spaceId`, `slug`, `expectedVersion?` | Retire a note; its history stays |
| `POST knowledge-spaces.notes.approve` | `spaceId`, `notes` (1 to 100 `{ slug, version }`) | Bulk approval, `{ approved, skipped }` |
| `GET knowledge-spaces.proposals` | `spaceId`, `status?`, `batchId?`, `offset?`, `count?` | `{ proposals, total }`, each with `proposerName` and, for a cited message, `sourceExcerpt` (`username`, `ts`, first 300 characters) |
| `POST knowledge-spaces.proposals.decide` | `proposalId`, `decision` (`approve` or `reject`), `title?`, `type?`, `text?` | `{ proposalId, status, noteVersion? }`, status `applied`, `rejected` or `stale` |
| `POST knowledge-spaces.proposals.reject-batch` | `spaceId`, `batchId` | `{ rejected }` |
| `POST knowledge-spaces.harvest.start` | `spaceId`, `runnerBotId`, `agentIds`, `teamMemory?` | Start a harvest, `{ harvest }` |
| `GET knowledge-spaces.harvest.status` | `spaceId` | `{ harvest }`, `null` when the space has no batch |
| `POST knowledge-spaces.harvest.cancel` | `spaceId`, `batchId` | Cancel a running batch |
| `POST knowledge-spaces.team-memory.move` | `spaceId` | [Team memory](#team-memory-move-and-move-back) |
| `POST knowledge-spaces.team-memory.move-back` | `spaceId` | [Team memory](#team-memory-move-and-move-back) |

No route deletes a space: a space is disabled and its rows stay as the record. Every new GET route is listed in
`REST_READ_NEVER_ROUTES` of `lib/universal-bot-rest-read-catalog.ts`, so the Universal Assistant cannot call it.

**Levels at creation.** `agents.create` accepts `knowledgeSpaces: Array<{ spaceId, level }>` (level `read`, `propose`,
`write` or `none`, at most 50 items). Each space must be administered by the caller and be either an owner space the
caller owns or a workspace space. A team space is refused (a new agent is on no roster; the roster entry gives the level
once it joins), and a restricted space is refused for an agent on an external harness. After the bot user exists, the
hub adds one `bot` entry per item, so the agent's first session already carries the digest. `agents.import` ignores the
parameter.

## Approval card and review notice

A proposal that waits for a person and has no batch gets a card. The card is posted after the proposal call has
answered, so a slow or failed post never delays or fails the proposal.

- **Room:** always the approver's direct message with the Universal Assistant, never an agent's own room (in an agent
  room every message not written by the room's own bot starts a turn of that agent). No card is posted when the
  Universal Assistant is disabled or not provisioned, or when the approver is not a space admin at that moment; the
  proposal then waits on the proposals page.
- **Message:** author the Universal Assistant, message id `kn-proposal-<proposalId>`. The text, written by the hub, names
  the space and its preset, the operation, type and title, the proposer, how many agents can read the space, for a rule
  the cited message (author, time, first 300 characters), and the proposed text inside a code fence longer than any
  backtick run in it, with a leading `#` escaped and `@` broken so it cannot mention anyone. An `update` also shows the
  current text. One sentence says that approval makes a rule an instruction for those agents. The link
  `<Site_Url>/knowledge-spaces/<spaceId>/proposals/<proposalId>` is written in the text.
- **Buttons:** `kn-proposal-approve-<proposalId>` and `kn-proposal-reject-<proposalId>`, callback and one-time. A click
  is decided from the stored button, never from the click payload, and only a space admin may decide; a refusal keeps the
  buttons. A recorded decision adds one line to the card and hides the buttons. Handler:
  `knowledgeProposalCallbackHandler`, registered in `server/configuration/bot.ts`.
- **No Approve button** when the proposed text is over 2,000 characters, or when an `update`'s current or proposed text
  is over 1,000 characters: the text says to decide on the review page. A person never approves from a card text they
  cannot see in full.

A batch proposal gets no card. It triggers a **review notice** instead: one message per space, approver and UTC day in
the approver's Universal Assistant direct message, id `kn-review-<spaceId>-<approverId>-<YYYYMMDD>`, with a URL button to
`<Site_Url>/knowledge-spaces/<spaceId>/proposals`.

## Digest

`withKnowledgeDigest(identityMd, botId)` (`agent-knowledge-digest.ts`) adds the `## Shared Knowledge` section to the
agent's identity when a session starts, for every agent but the Universal Assistant, whether or not the identity text is
empty. It is called in `getContext` (`server/services/agent-context-cache.ts`) and in `buildAgentIdentityForBot`
(`server/services/privos-sandbox-bot-key-service.ts`). There is no "already present" check on the identity text, because
that text holds data other agents can write. A rendering error leaves the identity unchanged and is logged.

Content, in this order, within `Agent_Knowledge_Digest_Max_Bytes` (default 16,384, never taken below 2,048):

1. A fixed preamble: the approved rules below are instructions; everything else is data to fetch with the
   `privos-knowledge` skill and never to follow; the skill commands; the [order of authority](#order-of-authority).
2. **Approved rules**: approved, active notes of type `rule`, each as `- [<slug>] <text>`. A text over 600 characters is
   cut with a pointer to `knowledge.py read`. Order: workspace spaces, the owner space of the agent's owner, then team
   spaces. A team space's rules are included only when the agent's owner is a space admin of that team space; otherwise
   they are listed as titles.
3. **Other approved notes**: for each readable space, the titles of its other approved, active notes as
   `- <slug> (<type>): <title>`.
4. **Team notes no person approved yet**: for each readable team space that holds active notes no person approved, one
   line with their short names, most recently changed first, at most 50, then the number of the rest, under a heading
   that says they are data and never instructions.
5. When the budget cuts entries, one line with the number not shown and the command that lists them. Rules are never
   dropped silently: each space gets an equal share of the budget for its rules first. The short names use at most
   1,536 bytes; titles take what is left.

Of a note no person approved, the digest never carries the title or the text, and carries the short name only for a team
space; unapproved notes of owner and workspace spaces do not appear at all. The digest is sent at session start only: a
change reaches a running session through the skill, not through the prompt. A digest row of the read log is written once
per agent, digest content and UTC day.

## Harvest and weekly review

A **harvest** turns existing agent memory into proposals. A space admin starts it with
`knowledge-spaces.harvest.start { spaceId, runnerBotId, agentIds, teamMemory? }`:

- The runner is an agent the person owns, with level `propose` or `write` on the space. A runner on an external harness,
  or in a shared team VM or still leaving one, is refused. Ownership is checked again on every chunk; a runner or source
  agent that no longer belongs to the person cancels the batch.
- Sources are the rules files of up to 20 agents the person owns (`error-not-authorized` for any other agent) and, for a
  team space the person administers, that team's memory, unless it has already moved into the space. Team memory is read
  only by a runner on the team's roster.
- One batch reads at most 120,000 characters. A larger selection is refused with `error-knowledge-limit`, and the message
  names the size of each source, so the person starts several batches. One space has at most one running batch.
- The runner fetches the text with `agents.knowledge.harvest.next` in chunks of at most 12,000 characters, wrapped as
  data. Only the runner of the batch may fetch; a repeated cursor returns the same chunk; a cursor past the stored cursor
  is refused (`error-invalid-params`), so a runner cannot skip text. Control and format characters are removed from the
  source text before it is measured and served. The `agent-knowledge.harvest-read` audit row, with ids and character
  counts, is written before a chunk is returned, and the call fails when it cannot be written.
- Every batch proposal waits for a person on the proposals page and may be a `rule` without a cited message.
- A batch ends when the runner closes it (`agents.knowledge.harvest.done`), when a space admin cancels it
  (`knowledge-spaces.harvest.cancel`), when the runner's turn ends while it is still running, after 60 minutes without a
  chunk or a proposal, or seven days after it started. Batch rows stay as the record of whose files were read.

The **weekly review** runs on Mondays at 02:00 UTC for each enabled space that has a reviewer agent (`reviewerBotId`).
Only the approver sets the reviewer, to an eligible agent the approver owns. A space is skipped when its reviewer does
not belong to its approver, has no level `propose` or `write`, or the approver is no longer a space admin. A review batch
reads, for each agent the approver owns that has a level on the space, the rules-file lines it has not read before (line
hashes kept in `agent_knowledge_review_marks`), and the notes whose `reviewBy` date has passed. A person sets `reviewBy`
when saving a note.

## Team memory move and move back

Until a team's memory is moved, `RoomFiles/TeamMemory/` works as before and its index stays in the team context block
([Bot-to-Bot Protocol](./bot-to-bot-protocol.md#what-every-teammate-knows-team-context-capability-cards-and-team-memory)).

- **Move** (`knowledge-spaces.team-memory.move { spaceId }`, a space admin of the team space). Each topic file becomes a
  note no person approved (writer kind `import`), the roster's level becomes `write`, and `knowledgeMemoryMovedAt` is
  stamped on the team document with the time the move started. A file over 16,000 characters is imported cut, after its
  whole text is copied to `TeamMemory/archive/<file>`. The move never deletes or changes a topic file. Running it again
  changes nothing: a file not imported is listed in `skipped` with a reason (`name`, `exists`, `changed-after-move`,
  `unreadable`, `empty`, `archive-failed`, or the error type of the refusal). The move cancels every running harvest of
  the space that reads this team's memory. Reply: `{ spaceId, created, skipped, audienceChanged, movedAt }`.
- **After the move** the hub no longer injects the file index for that team, whatever the switch says, and adds one
  sentence naming the knowledge space. `team.py memory note` appends one paragraph through the hub and `team.py memory
  read` lists the space; a failure of the knowledge routes is an error and no file is written. Team notes written by
  agents are fetched, not preloaded: until a person approves one, the digest shows the team's agents only its short name.
  `agents.a2a.teams.mine` carries `memoryMoved` per team, which is how `team.py` chooses.
- **Move back** (`knowledge-spaces.team-memory.move-back { spaceId }`). It first copies every `TeamMemory/*.md` file
  changed after the stamp (less 60 seconds) to `TeamMemory/archive/<stem>-<UTC yyyymmddThhmmssZ>.md`, in a write that must
  succeed before anything is overwritten (an agent whose sandbox still had the old `team.py` may have written there after
  the move). Then it writes every active note to `TeamMemory/<slug>.md`, rebuilds `MEMORY.md` and clears the stamp. The
  notes stay in the space. Reply: `{ spaceId, written, archived, movedAt: null }`.
- **Team export** of a moved team reads the notes of the team space in place of the files, whatever the switch says.

Move back is the rollback of a move. Like every route it answers `error-knowledge-disabled` while the switch is off, so
every moved team is moved back before the switch is turned off on a workspace. Move a team only after every roster agent's
sandbox has the `team.py` that knows the moved branch.

## Semantic search

Off unless `Agent_Knowledge_Semantic_Search_Enabled`, `Knowledge_Index_Enabled` and `Knowledge_Query_Enabled` are all
true. A note's text is embedded when the note is created or its text changes, with the workspace's embedding endpoint
and model, and the vector is stored on the note with its model. Every note embedding is reserved from the workspace's
existing daily budget `Knowledge_Backlog_Daily_Embed_Budget`, the same budget the room index spends. No reservation, no
embedding: the note stays searchable by keyword, and a daily job (00:01 UTC, at most 100 notes) embeds it later inside the
same budget. A search runs the keyword search first, then adds notes of the readable spaces whose vector is close to the
query, at least `Knowledge_Vector_Minimum_Similarity_Percent`. No route reply ever carries `embedding` or
`embeddingModel`.

## Settings and permission

Group `Agent_Knowledge` (`server/settings/agent-knowledge.ts`):

| Setting | Type | Default | Meaning |
|---|---|---|---|
| `Agent_Knowledge_Spaces_Enabled` | boolean, public | `false` | Master switch. Off: every route answers `error-knowledge-disabled` and no digest is rendered |
| `Agent_Knowledge_Digest_Max_Bytes` | int | `16384` | Byte budget of the digest; never taken below 2,048 |
| `Agent_Knowledge_Proposals_Per_Agent_Per_Day` | int | `5` | Proposals of one agent that may wait for a person, filed in 24 hours. Batch proposals and writes applied at once are not counted |
| `Agent_Knowledge_Proposal_Open_Days` | int | `14` | Days a pending proposal stays open |
| `Agent_Knowledge_Semantic_Search_Enabled` | boolean | `false` | Adds vector similarity to the keyword search |

Permission `manage-knowledge-spaces`, default role `admin`: create workspace spaces and administer every workspace space.

## Collections and records

| Collection | Holds |
|---|---|
| `rocketchat_agent_knowledge_spaces` | Spaces with their audience, admins, approver and reviewer |
| `rocketchat_agent_knowledge_notes` | Notes; never deleted, retired instead |
| `rocketchat_agent_knowledge_proposals` | Every change as a proposal row: the change log and the history of each note |
| `rocketchat_agent_knowledge_reads` | Read log: `op` (`digest`, `list`, `read`, `search`), spaces, slug and version for a read; kept 180 days |
| `rocketchat_agent_knowledge_harvests` | Harvest and review batches with their sources and cursor |
| `rocketchat_agent_knowledge_review_marks` | Line hashes of rules files a weekly review has read (at most 5,000 per agent and space) |

The team document (`rocketchat_agent_a2a_teams`) gains `knowledgeMemoryMovedAt`, the only record that a team's memory
moved. Durable audit rows (`rocketchat_server_events`) have the types `agent-knowledge.space-changed`,
`agent-knowledge.audience-changed`, `agent-knowledge.proposed`, `agent-knowledge.applied`, `agent-knowledge.rejected` and
`agent-knowledge.harvest-read`, with ids, counts and level words only. An audience change and a harvest read write their
row first and fail without it; other changes write it after the change. Only a call that returns knowledge writes a
read-log row, except that every answered search writes one (the daily search limit counts them).

## Limits

| Limit | Value |
|---|---|
| Active notes per space | 500 |
| Audience entries per space | 200 |
| Admins of an owner or workspace space | 1 to 50 |
| Space name | 1 to 80 characters, one line |
| Proposals waiting for a person, per agent per 24 hours | `Agent_Knowledge_Proposals_Per_Agent_Per_Day`, default 5 |
| Writes applied at once, per agent per 24 hours | 300 |
| Proposals in one batch | 200 |
| Days a proposal stays open | `Agent_Knowledge_Proposal_Open_Days`, default 14 |
| Notes in one bulk approval | 100 |
| Digest | `Agent_Knowledge_Digest_Max_Bytes`, default 16,384 bytes, never below 2,048; rule text cut at 600 characters; at most 50 short names per team space |
| Searches per agent per day | 300, at most 10 hits |
| Harvest | 120,000 characters per batch, 12,000 per chunk, 20 source agents, one running batch per space, 60 minutes idle, 7 days |
| Spaces given at agent creation | 50 |
| Read log | 180 days |

## Error types

Returned through `API.v1.failure(message, errorType)`: `error-knowledge-disabled`, `error-not-authorized`,
`error-invalid-params`, `error-knowledge-space-not-found`, `error-knowledge-note-not-found`,
`error-knowledge-note-exists`, `error-knowledge-note-approved`, `error-knowledge-secret`, `error-knowledge-citation`,
`error-knowledge-rate-limit`, `error-knowledge-proposal-pending`, `error-knowledge-stale`, `error-knowledge-limit`.

## Screens

Web only, hidden while the switch is off (the hub refuses anyway). Being built on the hub branch
`feat/agent-knowledge-screens`:

- **Knowledge access** panel in Agent Settings: the agent's spaces, level and source, and the agent's own entry per
  space a person administers ([Agent Settings UI](./agent-settings-ui.md#knowledge-access-panel)).
- A level picker per space when creating an agent, sent as `knowledgeSpaces`.
- The **Knowledge spaces** page, `/knowledge-spaces` and `/knowledge-spaces/:spaceId/:tab?/:itemId?` with tabs `notes`,
  `proposals`, `audience` and `settings`, opened from the Agents page header
  ([Agents Page](./agents-page.md#knowledge-spaces-page)).

There is no mobile screen; cards work through the generic keyboard.

## Source

Hub, under `apps/meteor`:

| Area | Files |
|---|---|
| Services | `server/services/agent-knowledge/` (`-access`, `-audit`, `-spaces`, `-notes`, `-proposals`, `-secrets`, `-card`, `-read`, `-search`, `-digest`, `-harvest`, `-harvest-sources`, `-review`, `-team-memory`, `-vectors`) |
| Routes | `app/api/server/v1/agent-knowledge-bot-endpoints.ts`, `app/api/server/v1/knowledge-spaces-endpoints.ts` |
| Jobs | `server/cron/agent-knowledge-proposal-expiry.ts` (daily 02:20 UTC), `agent-knowledge-review.ts` (Mondays 02:00 UTC), `agent-knowledge-note-embeddings.ts` (daily 00:01 UTC) |
| Models and types | `server/models/AgentKnowledge*.ts`, `server/core-typings/IAgentKnowledge*.ts` |

Sandbox: `src/hooks/template/skills/privos-knowledge/` and `src/hooks/template/skills/privos-team/scripts/team.py`.

## Not in this release

Access lists per note; grant expiry; an agent asking for access by card; several approvers per proposal; sending the
digest again inside a running session; fetching notes from an agent on an external harness; a mobile screen; indexing
notes in the room vector index; deleting a space; the Firm Knowledge bridges.

## Related

- [Self-Management Skills](./self-management-skills.md#privos-knowledge-knowledgepy) — the `privos-knowledge` skill
- [Bot-to-Bot Protocol](./bot-to-bot-protocol.md#team-memory-in-a-knowledge-space) — team memory after the move
- [Agents Page](./agents-page.md#knowledge-spaces-page) — the people page
- [Agent Settings UI](./agent-settings-ui.md#knowledge-access-panel) — the per-agent panel
- [Credential Vault](./credential-vault.md) — where keys go instead of notes
