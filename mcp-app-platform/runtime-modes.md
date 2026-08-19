# Runtime Modes — one app, three deployments

You write **one** app. The version that runs on the App Cluster (marketplace /
managed), the version a customer self-hosts over the Relay (standalone), and the
version you run on your laptop (development) share the **same manifest, the same
tools, and the same permission contract**. The only things that differ between
them are **transport** and **trust bootstrap**, and both are owned by the SDK —
not by your app code.

## The invariant

| Surface | Managed | Standalone-production | Development |
|---|---|---|---|
| Manifest (`privos-app.json`) | identical | identical | identical |
| Tools + handlers | identical | identical | identical |
| Permission contract / scopes | identical | identical | identical |
| **Transport** | Direct HTTP `/mcp`, cluster-brokered | Relay WebSocket (outbound) | Direct HTTP `/mcp` (loopback) or app-local dev Relay |
| **Trust bootstrap** | workload identity socket → signed dispatch | paired identity file → signed dispatch | none (unsigned, loopback only) |
| Caller identity (`context.actor`) | portal-brokered provenance | TOFU + Hub-signed user token | unverified |

Everything in the first three rows is your app. Everything in the last three is
the SDK's job.

## One entrypoint: `serveApp`

`@privos_ai/app-server` exposes a single entrypoint that resolves the mode and
wires the correct transport, trust bootstrap, and agent-bot Hub client
internally. Your app never imports `connectRelay`, `loadStandaloneIdentity`, the
identity controller, readiness checks, or the workload broker directly.

```ts
import { serveApp } from '@privos_ai/app-server';

await serveApp({
  descriptor,                       // your one AppDescriptor (built from privos-app.json)
  ui,                               // optional UI resource
  createHandler: ({ mode, agentBotHub, workloadIdentityClient, resolveHubOrigin }) =>
    buildMyHandler({ agentBotHub, workloadIdentityClient }),
  port,                             // explicit > HTTP_PORT > PORT > 8080
  resolveManifest,                  // standalone /ready input; default ${cwd}/privos-app.json
  configure: (app) => { /* /public, /ui, diagnostics — mounted BEFORE the MCP router */ },
});
```

`serveApp` resolves the mode with `resolveRuntimeMode()`:

- **managed** — a workload identity socket is present. Precedence: highest.
- **standalone-production** — a paired standalone identity file is present.
- **development** — neither is present, and `NODE_ENV` is not `production`.
- **Ambiguous** (both present) or **production without either** → a fatal
  `RuntimeModeError`. `serveApp` **fails closed**; it never silently guesses.

### `createHandler` context

- `agentBotHub` — the app's installation-bot Hub REST transport, already wired to
  the right Hub-origin source for the mode. Same `RoomBoundHubClient` shape
  everywhere.
- `workloadIdentityClient` — the managed workload identity **singleton** (managed
  mode only; `undefined` otherwise). Use this exact instance for `forRoom(...)`.
- `resolveHubOrigin()` — the Hub's own origin (managed: broker; standalone: paired
  file; development: `PRIVOS_URL`).

## Trust differences that stay visible (by design)

`serveApp` hides plumbing, **not trust**. These remain the developer's / operator's
concern and are the *only* differences left between modes:

- **Standalone pairing ceremony** — `serveApp` never creates or rotates an
  identity file and never triggers pairing. Pair out-of-band with `npm run pair`;
  it prints an **SSH-host-key-style fingerprint** (`SHA256:…`) for the operator to
  verify (TOFU). `serveApp` surfaces `StandaloneIdentityError` /
  `RuntimeModeError` verbatim if the file is missing or invalid.
- **Managed provenance** is portal-brokered; **standalone** provenance is
  TOFU + an admin-approved capability ceiling + an out-of-band fingerprint check
  + NTP/JWKS reachability. Managed dispatch assertions carry a 30-second budget
  with zero headroom — **NTP is a hard requirement** on both ends.

## Health and readiness

`/health` and `/ready` exist in **all** modes (marketplace Docker `HEALTHCHECK`
and the cluster monitor probe them):

- `/health` — liveness, always `200` while the process is up.
- `/ready` — mode-aware: managed reports workload-capability readiness, standalone
  reports the SDK readiness check (identity + Relay-authenticated + manifest
  lint + digest drift), development is a trivial `200`.

## Development transport

Development defaults to Direct HTTP on **loopback** (`127.0.0.1`) — unsigned
traffic must not leak off-host; bind `0.0.0.0` only with an explicit `host`.
`transportOverride: 'relay'` (the API form of `PRIVOS_TRANSPORT=relay`) is a
**development-only** affordance that steps `serveApp` aside for an app-local
interactive Relay pairing loop; any value under managed or standalone-production
is a boot error.

## Scripts

One convention: **`pair`** (out-of-band trust bootstrap) and **`start`**
(`serveApp`, mode auto-resolved). There is no per-mode start script.
