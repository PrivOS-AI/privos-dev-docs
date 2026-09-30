# Install and operate your own MCP app

Audience: a workspace admin or hub operator who installs a schemaVersion-3 MCP app
straight from its `privos-app.json`, and the developer who runs that app's server.
Both follow the same workflow, on a standalone hub or in a hosted workspace.

You run the app server yourself and it connects to the hub over Relay. The security
contract is the one a marketplace install has: signed hub dispatch assertions,
permission-catalog enforcement, a pinned manifest digest, per-app environment config,
an agent bot, and cleanup on uninstall. Installing this way does not publish anything
and does not replace marketplace review for apps distributed through the marketplace.

For the developer's side of the same flow (scaffold, `npm run pair`, `npm run dev`),
see the [Developer Guide](./developer-guide.md#4-develop-over-relay). For the admin
screens and API routes, see the [Admin Guide](./admin-guide.md).

## Enable the capability

The admin setting **`MCP_Standalone_V3_Install`** is on by default. It is admin-visible
and applies without recreating the container. A hub that has never stored a value adopts
the default; a hub that already stores a value keeps it.

Turning it off hides the **new-install** surface: neither its API route nor its UI is
available. Existing installs keep working, and their Settings → Uninstall control stays
available, so the switch can never strand an app.

## Get the app into the workspace

There are two entry points; both end with the same approval step.

- **Install from the manifest.** An admin with the `manage-oauth-apps` permission calls
  `POST /api/v1/mcp-apps.installFromManifest` with the app's `privos-app.json` as `manifest`
  and the permissions they allow as `approvedPermissionCeiling`. The hub validates the
  manifest digest and checks every declared scope against the permission catalog; an unknown
  or rejected scope refuses the install with an actionable error instead of installing a
  partly authorised app. Your approved ceiling is the authorisation: the hub issues the
  receipt itself and pins the manifest digest to the app. Next, open the app's settings and
  generate a pairing URL for it, so the app server can connect.
- **Pair first.** In Admin → Apps choose **Add Standalone Relay App**. The hub creates a
  pairing URL at once. The app announces its manifest when it pairs, and you approve the
  permissions it announces (required permissions cannot be cleared).

The app is a relay app (or a `direct` app if you supply a server URL) with execution mode
`PUBLISHER_HOSTED`. You host the app server; the hub never runs it.

## Pair the app server, and verify the fingerprint

The pairing URL has the shape `wss://<hub>/api/v1/mcp-apps.relay?pair=<token>` and is
valid for one hour. The route that creates it, `POST /api/v1/mcp-apps.generate-pair-url`,
returns the URL, its token and the **hub fingerprint** together. The admin screen shows the
URL only. Any signed-in user can also read the fingerprint from
`GET /api/v1/mcp-apps.standalone.fingerprint`.

The developer runs `npm run pair` in their own terminal and pastes the URL. The app
registers and prints the fingerprint it received, then waits until an admin approves its
permissions, and then starts. One run covers all of it; there is no second pairing URL.

**Check the fingerprint out of band.** The fingerprint the app prints must match the one
from the admin side, compared over another channel, such as a call or a separately verified
chat. This is the same trust decision as accepting an SSH host key: if you skip it, you
are trusting whatever answered the network.

Pairing hands the app everything it needs to verify this hub: the hub's public key, its key
id, and the generation-scoped affinity values. Until approval the app holds no dispatch
trust. If the first response is lost, repeat the exact same pairing URL with the exact same
manifest within 24 hours: a compatible hub returns the same registration and never creates
or rotates a second credential. A changed URL, manifest, permission ceiling or hub key is
refused.

### The identity file

The app stores its pairing result in one identity file (default
`./privos-standalone-identity.json`, mode `0600`; override with
`PRIVOS_STANDALONE_IDENTITY_FILE`). It refuses to load a file with loosened permissions.

**Keep the identity file at an absolute, persistent path outside your build output, and run
one process per app id.** The hub owns your app's dispatch trust: it rewrites this file on
approval and on every relay reconnect (the SDK verifies each rewrite against the pinned hub
key), so the file's whole job is to hold the current approval. If a redeploy copies a stale
identity file over the live one, or the process starts from a different working directory
than the default `./`, the app runs on an old approval and every dispatch fails with
`Authenticated private dispatch required` until the next reconnect heals it. Never copy,
commit, or bake the identity file into a deploy artifact; point
`PRIVOS_STANDALONE_IDENTITY_FILE` at a path on a persistent volume that your build never
overwrites.

## Two things that will bite you

**The hub identity file is the trust root.** It lives at `MARKETPLACE_INSTANCE_KEY_PATH`
(default `/var/lib/privos/marketplace/instance-identity.json`, mode `0600`). Back it up. If
you lose it, every paired app must be re-paired. Confirm the file is inside your backup
scope.

**Clocks must be right.** The hub signs each dispatch assertion with a 30-second lifetime and
the verifier accepts no longer lifetime than that. Skew tolerance defaults to 5 seconds and
applies only to a future-dated issue time. Run NTP on the app host, or dispatch will fail
intermittently and look like a network problem.

## Routine operations

**Rotate the relay secret** when a credential may have leaked (the rotation action is in the
app's settings). The old secret stays valid until the app's first successful reconnect with
the new one, capped at 24 hours, so in-flight work is not dropped. The new secret is
delivered to the app itself and is never shown to the admin.

**Update your app's manifest (standalone / relay).** Edit `privos-app.json` and restart the
app; it echoes the manifest it loaded at start. From the restart until approval the app's
`/ready` reports `MANIFEST_DRIFT` (503) by design, so do not let an orchestrator restart-loop
on that probe during an update. In Admin → Apps → *your app* → Settings click **Refresh**: the
hub compares the running manifest with the approved one and shows what changed (version,
tools, permissions). Tick the permissions you allow and **Approve update**; the new contract
is pinned, tools and entry points are updated, trust and the grant are pushed while the app
is online, and `/ready` becomes ready. If the app was offline at that moment the dialog keeps
a warning; use **Redeliver trust** once it reconnects. **Discard** keeps the approved
contract. Pairing again with a live app is refused and the refusal points you back to
Refresh; uninstall and re-pair is never required for a manifest change, and would drop room
bindings and the agent bot. Direct-connection (non-relay) standalone apps are not covered by
Refresh.

**Re-key the hub identity** only as disaster recovery, for example when the identity file was
lost. New trust is pushed to apps over their authenticated relay sessions. An app that was
offline during the re-key refuses the new key with `runtime_dispatch_trust_invalid` until you
re-pair it. That refusal is correct and deliberate: an app that silently accepted a new
signing key on reconnect would accept an attacker's key just as readily.

## Uninstall and reinstall

**Uninstall is one button, the same as for a marketplace app.** Settings → Uninstall → type
UNINSTALL → Uninstall permanently. Retrying the click is safe: a retried uninstall converges
on the same operation rather than cleaning up twice. Uninstall is never gated by
`MCP_Standalone_V3_Install`.

Uninstall destroys the runtime generation, all room bindings, the operator env config
(including encrypted secrets), the execution/agent bot and its agent room, and the relay
OAuth client together with every access and refresh token minted under it. The app then
disappears from the admin list.

The catalog row is retained (status `suspended`, stored manifest intact) for audit and
reinstall reuse. The admin apps list shows it when you enable **Show uninstalled apps**; the
same view is `GET /api/v1/mcp-apps.list?includeUninstalled=true`.

Return through a **fresh manifest pairing**, not by re-approving the retired pairing. The hub
reuses the one retained app row and mints a new zero-permission OAuth identity. The retired
client, tokens, generation and installation never return. Delete the old identity file on the
app host before you run `npm run pair` again, or pairing stops with
`IDENTITY_FILE_ALREADY_EXISTS`. Any pairing while a live generation exists is refused.

**An app id is live once per workspace.** A live Relay copy, or a registration that was
paired but never approved, makes a marketplace install of the same app id fail with
`mcp_app_id_conflict`. A fully uninstalled Relay copy is taken over by the marketplace
install. So uninstall the Relay copy before installing the same app from the marketplace.

**Delete is a different, narrower action.** `mcp-apps.delete` refuses an app that still has a
live generation (error code `error-active-generation-use-uninstall`); uninstall is the only
path once the app has been approved. Delete remains correct for a paired-but-never-approved
registration, an already-uninstalled app, or a legacy non-v3 app.

## Verifying it actually works

The app's `/ready` reports not-ready with a reason instead of guessing. The reasons are
`IDENTITY_NOT_LOADED` (the identity file must load), `RELAY_NOT_AUTHENTICATED` (the relay
must be authenticated), `MANIFEST_LINT_INVALID` and `MANIFEST_DRIFT` (the local manifest
digest must match the digest pinned at pairing).

Caller identity on the relay transport is verified: the hub mints a short-lived RS256 user
token per dispatch, the app verifies it against this hub's JWKS at
`/.well-known/mcp-apps/jwks.json`, and the token's room claim is cross-checked against the
room already bound in the signed assertion. The app host therefore needs to reach that JWKS
endpoint. If it cannot, the actor is reported as absent rather than assumed: verification
never degrades into trust. See
[Tools — Context › Signed user identity](./apis/tools-context.md#signed-user-identity).
