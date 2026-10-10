# Credential Vault

How PrivOS agents call external APIs without ever holding the API key. The sandbox proxy stores the secret encrypted,
injects it at egress for a matching binding, and the model only ever sees the response.

> **Status:** on by default. `VAULT_V2_ENABLED=0` (or `false`) on the proxy or the board is the operator kill switch,
> and the hub setting `PrivOSSandbox_Credential_Vault_Enabled` defaults to on for new installs (an upgraded hub keeps
> its stored value until an admin flips it). Without a valid `CATALOG_SECRET_KEY` (64 hex characters) the vault fails
> closed: it is not advertised, writes answer `vault-not-configured`, and agents are told that secure entry is not
> available instead of pausing. The hub shows vault features only when the proxy or board advertises them.

## Concepts

| Term | Meaning |
|---|---|
| **Binding** | `host` + `pathPrefix` + `methods` → one encrypted secret and a header or query template containing `{secret}` |
| **Scope** | Who owns a binding and which agents can use it: shared, room or agent-private |
| **Kind** | `vault` for owner-managed bindings, `platform` for the hub bot key the platform pushes. They live in one store, behind one `/egress` route |
| **Fingerprint** | First 8 hex characters of the secret's sha256. The only form of a secret any UI, log or audit row shows |

Secrets never enter the model's context, the container environment, files under `/run/privos-room`, chat messages or
logs. The proxy decrypts a secret only after a binding matches, injects it into the outbound request, and sends the
upstream response back to the skill.

## Scopes and who manages them

| Scope | Proxy scope id | Create, rotate, revoke | Used by |
|---|---|---|---|
| Shared (workspace) | `__tenant-global__` | Workspace admins | Every agent project in the workspace |
| Room | `room-scope:<roomId>` | The room's **owner** role and workspace admins. Moderators and leaders cannot | Every bot project bound to that room |
| Agent-private | `bot-scope:<botId>` | The bot's creator and workspace admins | That bot's projects in any room |
| Universal Bot | `bot-scope:universal-bot` | Workspace admins only | The Universal Bot's projects |

A room that runs its own sandbox has no room vault: its sandbox address is editable by moderators, so owner secrets
are never sent there. Use a shared or agent-private binding instead.

Bots and app users can never manage bindings. The hub refuses every vault route for a bot or app principal, and a bot
acting with its own bot key cannot reach `privos-sandbox.catalog.*` at all.

**Who can exercise a binding.** Be explicit about this when you create one:

- A room binding is usable by **every bot in the room**, including a bot a moderator or leader adds later.
- An agent-private binding is usable by **anyone who can drive that bot**: public rooms it sits in, on-behalf
  sessions, and other bots that delegate work to it.

Bind narrow path prefixes and the narrowest methods, and prefer provider-side scoped tokens (read-only, single
repository, single project).

## Binding reference

- **Host**: an exact host, or `*.registered-domain` (one label). Wildcards on public suffixes and on multi-tenant
  hosting domains are refused (`*.com`, `*.co.uk`, `*.vercel.app`, `*.github.io` and anything else on the Public Suffix
  List). IP literals, single-label hosts, the workspace hub and internal hosts are refused.
- **Path prefix**: a literal prefix such as `/repos/acme/`. A trailing `$` makes it exact. There are no globs in the
  middle of a path. Query strings never take part in matching.
- **Methods**: required, a non-empty subset of `GET HEAD POST PUT PATCH DELETE OPTIONS`. No method is allowed
  implicitly.
- **Template**: a header (for example `authorization: Bearer {secret}`) or a query parameter, with exactly one
  `{secret}`. Session headers (`x-auth-token`, `x-user-id`, `cookie`), `host`, `content-length`, `transfer-encoding`,
  `proxy-authorization` and any `x-privos-*` header are refused.
- **Scheme**: vault bindings are injected over `https` only.

### Which binding wins

1. The longest matching path prefix wins, whatever the scope.
2. On equal length the order is: platform, room, agent-private, shared.
3. Then the binding's `methods` are checked. A method outside the list is denied (`method-not-allowed-by-binding`);
   the proxy never falls back to a broader binding in another scope.
4. A room binding and an agent-private binding with the same host and path prefix are refused at write
   (`binding-conflict`). A tie that appears later, for example when a bot joins a room, is denied at match until an
   owner removes one side.

Exactly one binding or none is used; secrets are never merged.

## Lifecycle

- **Create** once. The value is write-only and never shown again; the UI shows the fingerprint.
- **Rotate** by entering a new value. The fingerprint changes and `version` increments. Methods and label stay unless
  you change them.
- **Revoke** deletes the binding. The next request is denied without a proxy restart.
- **Audit**: create, rotate, revoke and request events with actor, scope, pattern, version and fingerprint. Per-use
  metadata (time, scope, host, method, outcome) is kept separately and drives "last used".
- **Room services** that name a binding follow it: on a room-own sandbox a rotation restarts them and a revocation
  stops them; in a room VM the relay simply uses the new value or refuses the next call. See
  [Room Services](./room-services.md#credentials).

## Using it from a skill

A process that must outlive the agent's turn (a listener, a long job, a poller) names its bindings in a
[room service](./room-services.md) declaration (`vault: ["NAME"]`) instead of reading a key from a file.

Skills reach external services through the proxy's `/egress` route with the skill SDK's external client. The skill
never sees the key:

```ts
const res = await external.fetch('https://api.github.com/repos/acme/app/issues', { method: 'GET' });
```

When the proxy refuses a call, it fails with `EgressDeniedError` carrying `code`, `host`, a correlation `requestId`
and `canRequest` (true when a secret request may be filed):

| `code` | Meaning |
|---|---|
| `no-binding` | No binding covers this host and path |
| `method-not-allowed-by-binding` | A binding matched but not for this method |
| `binding-conflict` | A room binding and an agent-private binding tie; an owner must remove one |
| `insecure-scheme` | A vault binding matched a plain `http` URL |

### Asking a human for a credential

On `no-binding` with `canRequest`, a skill may file one request. The request carries only the host, an optional path prefix and a
one-line purpose. Never a value, scope or method:

```ts
const { requestId, status } = await vault.requestCredential({
  host: 'api.github.com',
  pathPrefix: '/repos/acme/',
  purpose: 'Read open issues for the weekly report',
});
```

Then tell the user and stop. Never ask the user to paste the key in chat. The bot's owner gets a card in their
Universal Bot DM with a link to the vault form; they choose the scope and methods there and enter the value once. The
agent retries later; `vault.requestStatus(requestId)` reports `pending`, `resolved` or `declined`. Requests are rate
limited per project and deduplicated while pending.

## Asking the user through askUser

An agent that needs a key asks for it with its `askUser` tool and **one** `credential` question: the host, an
optional path prefix, a one-line purpose, and suggested env names. It must be the only question and has no options.

1. The runtime files a request with the vault (`origin: ask-user`) and pauses the agent. If a vault binding already
   covers the host, nothing is filed and the agent is told to use it (`already-bound`). Hidden runs, collocated rooms,
   a disabled vault and rate limits each get a clear immediate answer, with no pause.
2. The hub posts a card in the thread built only from the vault request (registrable domain, full host, path prefix,
   purpose, who can complete it) with an **Open the secure form** button. The model's own question text is never
   shown, and the card says not to paste the key in the thread.
3. The person opens the form, picks the scope (agent-private by default; the room option warns that every bot in the
   room, including bots added later, can use the key), the methods and the env names, and enters the value once. The
   save settles **that** request by id; a binding that does not cover the requested host and path is refused.
4. The hub resumes the agent exactly once. The agent gets the host, path prefix and env names, never the value or
   its fingerprint.

Every other answer path refuses a typed answer for a credential question before anything is stored: a thread reply,
the AI Chat question form, an AI Chat reference answer, `agents.sandbox.answer`, voice and uploads. Separately, an
always-on guard refuses a human message in a bot thread or a bot DM that looks like a key (known key prefixes, JWTs,
PEM private keys, `token=`/`secret:` pairs with a long value). Commit SHAs, UUIDs and URLs pass.

If the attempt ends before the key is saved, the thread gets a notice; a key saved later is still stored. If no
thread claims a request within a minute (for example a board-direct attempt), the bot creator or room owner gets the
card in their Universal Bot DM.

**Rotation from the agent.** When a stored key is rejected (401 or 403), the agent asks again with `rotate: true`.
The card opens the form in rotate mode: value only, with scope, pattern, methods and env names read-only. The version
increments and the agent is told to retry. The Vault tab in the sandbox settings keeps its own create, rotate and
revoke actions.

The platform skill `privos-vault` teaches agents this flow, how to use the variables below, and never to echo them.

## Env variables, vault.env and the base-URL relay

A vault binding can carry two env names: one for the key (`^[A-Z][A-Z0-9_]{1,63}$` ending in `_API_KEY`, `_KEY`,
`_TOKEN` or `_SECRET`) and one for the base URL (ending in `_BASE_URL` or `_API_BASE`). Names starting with
`PROXY_`, `PRIVOS_`, `ROXANE_`, `CLAUDE_`, `AGENT_`, `PROJECT_`, `LD_`, `DYLD_`, `NODE_`, `PYTHON`, `GIT_`,
`BASH_`, `NPM_CONFIG_` or `PIP_`, and exactly `ANTHROPIC_API_KEY` and `ANTHROPIC_AUTH_TOKEN`, are refused.

The agent never gets the key. The key variable holds a **relay token** (`pvr_…`, one per project, accepted only by
the relay) and the base-URL variable holds `<relay>/vault-relay/<host><prefix>`. The relay authenticates the token
from a header (`Authorization`, `x-api-key` or `api-key`, never the query), matches a vault binding exactly like
`/egress`, swaps in the real key, and forwards over pinned HTTPS with no redirects and with echo masking. Responses
stream, so SSE works. Only bindings with an exact host get a base-URL variable.

- **Which tools work:** any SDK or CLI that honours a base-URL variable, for example the `openai` SDKs with
  `OPENAI_BASE_URL`. A tool that ignores base-URL variables cannot use the vault this way; in sandbox mode use
  `external.fetch` or `egress_request` instead.
- **Sandbox mode:** the proxy writes `/run/privos-room/vault.env` next to `skills.env` and rewrites it when a
  binding changes. Every shell command sources `skills.env` and then `vault.env`, so a vault name replaces a
  plaintext room variable of the same name. Stdio MCP servers get the file's variables at attempt start, so a key
  saved mid-attempt reaches MCP servers from the next attempt.
- **Collocated rooms** have no `vault.env` and cannot ask; they use `external.fetch` with shared bindings.
- **Precedence** when two visible bindings use the same name: platform, room, agent-private, shared. An equal-rank
  tie drops the name. Writes refuse a name that is already used in the same scope or in the shared scope.

## Board mode

A board without a sandbox proxy (`PRIVOS_SANDBOX_MODE` unset) runs the same vault module in-process on loopback,
with its own sqlite file and the same REST contract, so the hub's card, form and Vault tab work unchanged against the
board URL. The board uses `CATALOG_SECRET_KEY` from its env when set, and otherwise generates a key once into its data
directory (mode `0600`) and keeps it in memory only. A check value stored next to it makes the vault refuse to start
if a later start resolves to a different key, so stored keys are never silently orphaned. Claude CLI agents get the
vault variables in their process env (and their stdio MCP servers inherit them). A vault name replaces an inherited
or room `.env` variable of the same name, except reserved names and anything starting with `ANTHROPIC_`, which the
agent itself needs; a binding's key and base-URL names are applied or skipped together. The relay listens on the
vault's loopback port only, so a relay token is useless off the host, and it is revoked when the board project is
deleted. The vault admin routes accept the board API key the hub uses, never agent keys. The board's built-in agent runtime (the `privos-agent-sdk` provider) answers a credential
question with a clear refusal in board mode, because its shell and MCP servers do not receive vault variables there;
use the Claude CLI provider or sandbox mode.

**Intended use:** board mode is for private agents used by one person. Keys entered for them are not meant to be
shared with other people's agents. A room that points at its own sandbox never gets the secure form: its sandbox address
is editable by moderators, so the Hub sends keys only to the workspace sandbox. To use the vault with a board through a
Hub, pair the board as the workspace sandbox.

**Ceiling, accepted:** the CLI runs as the board's OS user, so a hostile agent on the same host can read the board's
key and database and decrypt stored keys. Chat, logs, transcripts and model context stay clean. Use sandbox mode when
agents must not be able to reach the keys at all.

## Coexistence with the bot key

| | Bot-key egress catalog | Credential vault |
|---|---|---|
| Purpose | Authenticate the agent to its own hub | Authenticate the agent to external services an owner chose |
| Rows | `kind: platform`, project scope, hub-host route patterns | `kind: vault`, shared, room or agent-private scopes, external hosts only |
| Writer | The hub's bot-key push | Hub vault routes (admin, room owner, bot owner) |
| Rotation | The hub re-issues the bot token | The owner rotates in the hub |

Both use the same proxy, the same `/egress` route, the same SSRF guard and the same encryption key. The bot-key push
never touches vault rows, and the vault UI never lists platform rows. See
[Bot Key & Agent Switching](./bot-key-and-agent-switching.md).

### Common mistakes

1. **An API key in a room MCP server header or a skill env variable** ends up in plain text inside the agent VM, where
   the agent can read it and send it anywhere it can reach. Anything that must stay out of the VM belongs in the vault;
   give the binding env names and the MCP server or SDK gets a relay token instead.
2. **Binding the hub host in the vault** is refused; the hub is reached with the bot key.
3. **A skill calling the hub through `external.fetch`** works: the platform binding applies and nothing leaks.
4. **A key in a `.env` next to a `nohup` or pm2 process** is a key in plain text outside the vault, and the process
   dies or lingers unseen. Declare a [room service](./room-services.md) that names the binding in `vault`; hub events
   need no process at all (filtered triggers).

## Threat model in brief

The design assumes the model is hostile.

- **Echo back:** if the bound API echoes the secret in a response body or header, the proxy masks it
  (`[redacted-by-vault]`) or cuts the stream, and redacts response headers that contain it.
- **Redirects:** redirects are never followed with credentials, and cross-host `Location` headers are stripped.
- **SSRF:** hosts are resolved and pinned, and internal and metadata addresses are denied.
- **Model writes:** agents cannot create or widen bindings.
- **Residual risk:** an API that mints new tokens can still hand the model a fresh, live token. Narrow methods and
  path prefixes are the control for that.

An optional network mode (`VM_EGRESS_MODE=enforce`) also takes direct internet access away from agent VMs. Uncredentialed
traffic then goes through the proxy's forward listener, and only to hosts an admin lists in
`PrivOSSandbox_Egress_Allowed_Hosts`.

## Related docs

- [Room Services](./room-services.md)
- [Bot Key & Agent Switching](./bot-key-and-agent-switching.md)
- [Architecture](./architecture.md)
