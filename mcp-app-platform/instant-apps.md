# INSTANT apps — frontend-only, no runtime

An INSTANT app is a PrivOS MCP app with **no server, no container, nothing
listening on a port**. What you build is a static UI bundle that the Hub
preloads into the workspace and serves to every user who opens it (see
[The signed UI bundle](./ui-bundle.md)). Application data lives in
**PrivOS Lists**, reached from the browser through the same
`@privos_ai/app-react` bridge every MCP app UI uses — there is just no backend
on the other end of any of its own tool calls.

## When to choose it

Choose INSTANT when your app is a UI over data PrivOS Lists can already model
(records, statuses, checklists, boards) and you have no need for a server-side
integration, background job, or third-party API call. Choose a runtime app
(managed or self-hosted) instead the moment you need any of that — INSTANT has
no path to add a backend later without becoming a different execution mode.

The [OKR Goals Book](https://github.com/PrivOS-AI/privos-okr-instant-app) is a
complete, real INSTANT app: a room-tab dashboard backed by three Lists, with a
worked manifest and a detailed README. Read its
[`docs/adding-instant-to-manifest.md`](https://github.com/PrivOS-AI/privos-okr-instant-app/blob/main/docs/adding-instant-to-manifest.md)
for a field-by-field walkthrough of its own `privos-app.json`.

## Scaffold

```bash
npx create-privos-mcp-app my-app --template instant
```

The `instant` template has no `server.ts` and no `Dockerfile` — there is no
runtime for either to configure. It ships a Vite + React UI under `src/ui/`
and a `privos-app.json` already set to `executionMode: "INSTANT"`.

## Adding INSTANT to the manifest

Every rule below is enforced by the SDK's lint (`manifest-lint-instant.ts`)
whenever `executionMode: "INSTANT"` is present — a manifest that omits it is
untouched by these rules.

### `executionMode`

```json
{ "executionMode": "INSTANT" }
```

### `ui.entryPoints`

Required. Declares which room surfaces render your UI. Each entry point needs
a `title` and a `resourceUri`:

```json
{
  "ui": {
    "entryPoints": {
      "roomTab": { "title": "My App", "resourceUri": "ui://com.example.my-app/dashboard.html" },
      "standalone": { "title": "My App", "resourceUri": "ui://com.example.my-app/dashboard.html" }
    }
  }
}
```

- **Slots:** `roomTab` (required), `sidebar`, `standalone` (both optional). Any
  other key is rejected.
- **`resourceUri` format:** `ui://<appId>/<file>.html` — `<appId>` must equal
  the manifest's own `name`, and `<file>.html` must match
  `[A-Za-z0-9][A-Za-z0-9._-]*\.html`.
- Multiple entry points may point at the same `resourceUri` — INSTANT ships one
  static bundle regardless of how many surfaces reference it (the OKR app's
  `roomTab` and `standalone` both point at `dashboard.html`).

### Fields an INSTANT manifest must never declare

There is no runtime to configure, so these are lint errors, not warnings:
`tools`, `serverUrl`, `runtimeTrustProvisioningUrl`, `port`, `resources`,
`volumes`, `stateless`. A `Dockerfile` left in the project directory is not an
error — it's simply ignored, since no image is ever built for a frontend-only
app.

### `agent` (optional)

An INSTANT app may still ship an agent persona for its room's AI Chat, with the
same size caps the Hub enforces server-side (a lint error here, not a silent
truncation, so you find out before publishing):

| Field | Cap |
|---|---|
| `purpose` (required) | 2000 characters |
| `personality` | 2000 characters |
| `instructions` | 5000 characters |
| `knowledge` | 20 items, 500 characters each |
| `hubTools` | array of non-empty tool names; the Hub narrows this further at install time against the workspace's own bot permission catalog — a manifest can only ask for a subset |

### `ui.shellMode`

Leave unset (defaults to `"static"`, the only mode INSTANT allows).
`"live"` is a lint error for `executionMode: "INSTANT"` — see
[The signed UI bundle](./ui-bundle.md#uishellmode).

## Data lives in PrivOS Lists

There is no database of your own. Model your app's data as one or more
PrivOS Lists, created on first use from the browser (`lists:write`) and read
back with `lists:read` / `lists:query`. See the
[React SDK reference](./react-sdk-reference.md) for the `useLists` hook, and
the OKR app's README for a worked three-list data model with a documented
find-or-create race and how it's resolved.

## Permissions

Declare only what the UI actually calls, same as any other app — least
privilege, `context`/`executionContext` per scope, and an optional
`degradedBehavior` string for anything marked `requirement: "optional"` so a
user who declines it still gets a coherent (if reduced) app instead of a
broken one.

## Build and bundle

```bash
npm run build           # your normal UI build (e.g. vite build → dist/)
npm run manifest:lint   # privos-app lint privos-app.json — structural checks
npx privos-app bundle-ui --check   # what publish-lint runs internally; --out to inspect the tar locally
```

The build node runs the same `bundle-ui` step at publish time — see
[The signed UI bundle](./ui-bundle.md) for what happens after that (Portal
signs the digest, the Hub verifies and preloads it at install/upgrade).

## Publish checklist

- **SDK ≥ 0.12.1.** `@privos_ai/app-server` 0.12.0 required a `Dockerfile` in
  every archive; INSTANT apps have none. Depend on `^0.12.1` or later.
- **Commit `package-lock.json`.** The build node runs `npm ci` against it.
- **Run `privos-app publish --dry-run` before publishing, not just
  `lint --publish`.** The plain lint has been seen to pass a manifest with
  duplicate permission `feature` ids that `publish` itself rejects — the
  dry-run catches it before you spend a real submission on it.
- **The marketplace listing must allow the INSTANT execution mode.** Creator
  Studio has no execution-mode editor yet; a new listing defaults to
  `PRIVOS_MANAGED_RUNTIME` and will refuse an INSTANT manifest until the
  listing's `supportedExecutionModes` includes `"INSTANT"`. Ask a marketplace
  admin if your listing isn't set up for it yet.
- **Listing content gates still apply** — categories, features, use cases,
  support/privacy/terms URLs, an approved icon, and captioned screenshots are
  listing fields, not manifest fields. See
  [Publishing CLI](./publishing-cli.md) and the demo app's
  [PUBLISHING.md](https://github.com/PrivOS-AI/privos-mcp-app-demo/blob/main/PUBLISHING.md)
  for the general listing-content requirements every app, INSTANT or not,
  goes through.

## Reference

[privos-okr-instant-app](https://github.com/PrivOS-AI/privos-okr-instant-app) —
a small, real INSTANT app with a worked manifest, a documented three-list data
model, and its own
[`docs/adding-instant-to-manifest.md`](https://github.com/PrivOS-AI/privos-okr-instant-app/blob/main/docs/adding-instant-to-manifest.md)
guideline.
