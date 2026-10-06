# Credential Vault

How PrivOS agents call external APIs without ever holding the API key. The sandbox proxy stores the secret encrypted,
injects it at egress for a matching binding, and the model only ever sees the response.

> **Status:** behind switches that default to off. The proxy needs `VAULT_V2_ENABLED`, and the hub needs the setting
> `PrivOSSandbox_Credential_Vault_Enabled`. The hub shows vault features only when the proxy advertises them.

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

## Using it from a skill

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
   the agent can read it and send it anywhere it can reach. Anything that must stay out of the VM belongs in the vault.
2. **Binding the hub host in the vault** is refused; the hub is reached with the bot key.
3. **A skill calling the hub through `external.fetch`** works: the platform binding applies and nothing leaks.

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

- [Bot Key & Agent Switching](./bot-key-and-agent-switching.md)
- [Architecture](./architecture.md)
