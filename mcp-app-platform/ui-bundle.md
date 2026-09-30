# The signed UI bundle

Any marketplace version that declares a UI — a `ui://` resource on a tool, or (for
an [INSTANT app](./instant-apps.md)) `ui.entryPoints` — ships with a **signed UI
bundle**: a deterministic tar of the built UI, produced once by the SDK and
verified independently by the Hub before that UI is ever served to a user. A
version with no UI (tools-only) has no bundle and skips every step below.

## What's in the bundle

`shell.html` + `assets-manifest.json` + `assets/<file>` — exactly what
`serveBuiltUi` reads and serves at runtime, packed into a USTAR tar built from
your UI build output (`ui.distDir` in `privos-app.json`, default `dist`). The
shell is rendered through the same code path `serveBuiltUi` uses, so the bundle
and what a running app would serve live are byte-identical.

Platform budgets, enforced before any file is written: **≤ 2 MB per file, ≤ 256
files per bundle, ≤ 64 MB total**. A build that exceeds one of these fails the
bundle step, not the Hub — fix it before publishing.

## `privos-app bundle-ui`

```bash
npx -p @privos_ai/app-server privos-app bundle-ui [--dist <dir>] [--out <file>] [--check] [--cwd <path>]
```

| Flag | Meaning |
|---|---|
| `--dist <dir>` | UI build output directory. Overrides `ui.distDir` / the `dist` default. |
| `--out <file>` | Write the deterministic tar to this path. |
| `--check` | Validate only (build + budget checks); never writes `--out`, even if given. This is what `privos-app lint --publish` runs internally. |
| `--cwd <path>` | Directory containing `privos-app.json` (default: current directory). |

Exit codes: `0` valid, `1` build or budget failure, `5` usage error.

You will rarely run this yourself outside of local testing — see
[Publishing CLI](./publishing-cli.md) for where it fits in `privos-app publish`.

## `ui.distDir`

```json
{ "ui": { "distDir": "dist/ui" } }
```

Set this when your build tool's output directory isn't the SDK default
(`dist`) — for example a Vite config with `build.outDir: 'dist/ui'`. Getting
this wrong doesn't fail loudly: `bundle-ui` runs against whatever directory it
resolves and bundles the wrong (or a stale/empty) set of files.

## `ui.shellMode`

`"static"` (default) or `"live"`. A static shell is rendered once at build
time and served byte-identical to every user — this is what the bundle
captures, and what `privos-app lint --publish` checks for accidental per-user
data (template placeholders, JWT-shaped tokens) baked into it. `"live"` opts a
runtime app out of shell bundling entirely so it can render a per-user shell
at request time; it is **not allowed** for an [INSTANT app](./instant-apps.md)
manifest, since there is no runtime to ever serve one.

## What happens at publish

Once a version reaches admin approval, the marketplace build node runs
`privos-app bundle-ui` (the same published SDK command above) over the built UI
directory. The Portal stores the resulting tar and records its sha256 digest on
the version's signed descriptor. A version whose manifest declares a UI but
whose build never produced one, or whose bundle fails the budget/shape checks,
does not reach `PUBLISHED`.

## What happens at install and upgrade

A capable Hub reads the signed digest off the version descriptor, pulls the
matching bundle, and verifies its sha256 against that digest **before** the app
is allowed to go active (install) or before the runtime swap (upgrade). Only
after that verification does the Hub store the UI in the workspace's own object
storage — from then on, the Hub serves that app's UI straight from workspace
storage; opening the UI never calls out to the app itself.

- **Manifest declares a UI but the version carries no signed bundle** — the
  marketplace refuses before anything is installed: an install shows
  `MARKETPLACE_V3_UI_BUNDLE_MISSING`, an upgrade to such a version shows
  `MARKETPLACE_V3_UPGRADE_TARGET_UI_BUNDLE_MISSING`. If a Hub still reaches its
  own gate without a bundle it refuses with `UI_BUNDLE_MISSING`. In every case
  no runtime work runs.
- **The pulled bundle fails verification** (digest mismatch, malformed tar, a
  disallowed entry) — the operation is refused with
  `UI_BUNDLE_INGEST_FAILED:<code>`. On an upgrade this is what the Hub reports
  as `UPGRADE_REFUSED_RETRYABLE`: the previous version keeps serving
  untouched, and retrying succeeds once the underlying bundle problem is fixed
  (a re-publish, usually).

This applies to apps that run on the PrivOS marketplace runtime, a self-hosted
runtime, and INSTANT apps alike — anything the Hub itself has to serve a UI
for.

## Identical UI, identical digest

The bundle is a deterministic build: the same UI source produces byte-identical
tar bytes, and therefore the same sha256 digest. A new version whose UI didn't
change reuses the workspace's already-stored generation instead of re-ingesting
it. To ship a UI change, change the UI source — a manifest-only or
backend-only version bump does not re-trigger UI ingestion.

## Tools-only apps

An app whose manifest declares tools with no `ui://` resource and no
`ui.entryPoints` has nothing to bundle. `bundle-ui` and the install/upgrade
verification gate above are both no-ops for it.

## Standalone relay apps serve live

A [Standalone Relay app](./admin-guide.md) — the developer runs their own app
server and connects over the relay — serves its UI live from that server on
every open. There is no signed bundle, and the Hub never stores that app's UI
in workspace storage.
