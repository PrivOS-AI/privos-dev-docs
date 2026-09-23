# Serving File Binaries in Apps (images, audio, downloads)

How to make a Hub **Files** binary (image, audio, PDF, …) loadable by a browser
— either inside an MCP app iframe or on an app-served public page. This is the
missing half of [`file-management-api.md`](./file-management-api.md): that doc
lists the *upload*, *metadata* and authenticated `/download` endpoints, but not
how a browser actually renders those bytes.

## The problem

A file's bytes live in the Hub's object store (MinIO). Three tempting paths all
FAIL from a browser:

| Path | Why it fails |
|------|--------------|
| `file.downloadUrl` from `GET /file-management.files/:id` | It is a **presigned URL to the Hub's INTERNAL object store** (e.g. `http://10.88.0.13:9000/...`). The user's browser cannot route to it. |
| `GET /file-management.files/:id/download` | Requires an authenticated bearer/`files:read`. An MCP app iframe runs in an **opaque origin** with no token or cookie, so it cannot call it directly. |
| Fetch bytes through the app REST proxy (`app.rest()`) | The Hub REST proxy reads the downstream response as **text** (`res.text()`), which **corrupts binary**. It is fine for JSON/text artifacts only. |

## The working endpoint

The Hub exposes an **unauthenticated inline content route**, served by the Hub
**web origin** (not MinIO):

```
GET /api/v1/file-management.files/:fileId/content/:filename
```

- `authRequired: false` — protected instead by the **unguessable 24-hex ObjectId**.
- The `:filename` segment is **cosmetic and ignored**; the `Content-Type` comes
  from the stored file. Any placeholder works (`.../content/image`).
- **Always include the `:filename` segment.** The no-filename variant
  (`/content` without a trailing segment) requires auth — do not use it.
- Streams the bytes with the right `Content-Type`, so an `<img>`, `<audio>`,
  `<video>` or a download link loads it cross-origin with no auth.

> This route is intentionally not in the endpoint summary of
> `file-management-api.md` because it is a *rendering* affordance, not a
> data-plane API. It is verified in production by the genealogy, business-hub
> and meeting-agent apps.

---

## Case A — Inside the app iframe (client)

The app iframe runs in an opaque origin and is **not told the Hub's public URL**.
Recover the parent (Hub) origin from `ancestorOrigins` (fallback: `referrer`),
then build the content URL. Using the *parent* origin also serves custom Hub
domains correctly.

```ts
/** The origin of the Hub page that embeds this app, or null when unknown. */
export function hubOrigin(): string | null {
  try {
    const ancestors = window.location.ancestorOrigins;
    if (ancestors && ancestors.length > 0 && ancestors[0]) return ancestors[0];
  } catch { /* ancestorOrigins unavailable in some engines */ }
  try {
    if (document.referrer) return new URL(document.referrer).origin;
  } catch { /* malformed/absent referrer */ }
  return null;
}

/** `<img src>` / `<audio src>` URL for a Hub fileId, or null when standalone. */
export function hubFileContentUrl(fileId: string | undefined | null): string | null {
  const id = typeof fileId === 'string' ? fileId.trim() : '';
  if (!id) return null;
  const origin = hubOrigin();
  if (!origin) return null; // opened standalone → caller falls back to a placeholder
  return `${origin}/api/v1/file-management.files/${encodeURIComponent(id)}/content/image`;
}
```

Usage:

```tsx
<img src={hubFileContentUrl(avatarFileId) ?? undefined} alt="" loading="lazy" />
<audio controls src={hubFileContentUrl(audioFileId) ?? undefined} />
{/* Download: a plain link (needs sandbox `allow-downloads` on the app frame) */}
<a href={hubFileContentUrl(fileId) ?? '#'} download={fileName}>Download</a>
```

Returns `null` when opened standalone (no parent origin) — render initials/a
placeholder instead.

### CSP note (matters for `<audio>`/`<video>`, not usually `<img>`)

The content route is on the **Hub web origin**. Most apps declare **no**
`ui.csp` block, so `img-src`/`media-src` fall back to the Hub's permissive
baseline and images just work (genealogy, business-hub).

If your app **does** declare a `ui.csp` block (e.g. because it needs
`connect-src` for a third-party WebSocket), then any directive you list is
enforced verbatim — it does **not** fall back to the baseline. So an app that
declares `media-src` MUST include the Hub web origin there, or `<audio>`/
`<video>` from the content route is blocked. `img-src` is only affected if you
declare it. Changing the manifest CSP triggers a one-time `MANIFEST_DRIFT` →
resolve with **Hub Admin → Apps → Refresh + approve**.

---

## Case B — On an app-served public page (server, no iframe)

A public HTML page rendered by the app's own server has **no parent origin** to
scrape (`ancestorOrigins` is a browser-only thing), and the SDK hides the Hub's
browser-facing host from server code. So you cannot embed a Hub URL directly.
Instead, the app serves the bytes from **its own origin** and streams them from
the Hub as the installation bot.

```ts
const OBJECT_ID_RE = /^[a-f0-9]{24}$/; // guard: block path injection into the Hub

async function handleMedia(req, res) {
  const fileId = String(req.params.fileId ?? '');
  if (!OBJECT_ID_RE.test(fileId)) return void res.status(404).end();
  if (!isHubChannelReady()) return void res.status(404).end(); // degraded → hide, not 500

  const upstream = await agentBotAuthorizedFetch(
    `/api/v1/file-management.files/${fileId}/content/image`,
    { method: 'GET', requiredScope: 'basic:information' }, // route is authRequired:false
  );
  if (!upstream.ok) return void res.status(404).end();

  const contentType = upstream.headers.get('content-type') ?? 'application/octet-stream';
  if (!contentType.startsWith('image/')) return void res.status(404).end(); // allowlist
  const buffer = Buffer.from(await upstream.arrayBuffer()); // arrayBuffer, NOT .text()
  res.status(200).setHeader('Content-Type', contentType);
  res.setHeader('Cache-Control', 'public, max-age=300'); // fileId is immutable; keep short
  res.end(buffer);
}
```

Then embed a **same-origin** URL (`/<your-public-path>/media/<fileId>`) in the
HTML. Key points:

- The content route is `authRequired: false`, so `requiredScope` can be the
  always-granted `basic:information` — installs that never enabled `files:read`
  still serve media. Still go through the bot fetch (the only channel to the Hub).
- Read `arrayBuffer()`, never `.text()` (binary corruption).
- Validate `fileId` as a 24-hex ObjectId **before** interpolating it into the
  path (path-injection guard).
- Allowlist the response `Content-Type` for a public route (serve only what you
  intend, e.g. `image/*`).
- On a not-ready channel or upstream error, return **404**, not 500 — the page
  still renders its text.

---

## Which case to use

| Context | Use |
|---------|-----|
| MCP app iframe (`<img>`/`<audio>`, in-app download) | **Case A** — content URL from `ancestorOrigins`. |
| Public page served by the app's own Express | **Case B** — same-origin proxy that streams via the bot. |
| Structured JSON/text artifact (transcript.json, .srt, .md) | Neither — read it through `app.rest(GET .../download)`; the proxy returns the parsed body (binary is corrupted, JSON/text is fine). |
| Large binary you must fetch server-side (concat, transcode) | Authenticated `GET /file-management.files/:id/download` as the bot; stream, never buffer whole. |

## Reference implementations

- Case A: `genealogy-privos-mcp-app/src/ui/data/room-file-url.ts`,
  `mcp-app-clan-business-hub/src/ui/features/media/hub-file-url.ts`.
- Case B: `mcp-app-clan-business-hub/src/public/public-media-proxy.ts`.
