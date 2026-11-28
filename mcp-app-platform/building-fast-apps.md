# Building fast apps: data in the browser, UI first, offline first

An app feels slow when the user waits for something they did not ask for: a trip through the app
server to read data the browser could read itself, or a setup check that holds the whole screen.
Three habits avoid that:

1. **Read List data in the browser**, through the host bridge, instead of loading it into your app
   server and handing it back.
2. **Draw the UI first.** Run checks in the background, and open a dialog when the user must know
   or decide something.
3. **Offline first.** Show the last copy at once and refresh it in the background, in the UI and in
   the app server.

The `create-privos-mcp-app` templates (`default` and `instant`) follow all three; start from their
`src/ui/App.tsx`, `src/ui/offline-cache.ts` and `src/ui/notice-dialog.tsx`.

## Why the path matters

Where a call goes decides what it costs:

| Call from the UI | Who answers | Path |
|---|---|---|
| `app.rest({ path: 'items.query', ... })` | The Hub | frame → host → Hub |
| `app.callServerTool({ name: 'mcpapp.lists.queryItems', ... })` (a Hub built-in) | The Hub | frame → host → Hub |
| `app.callServerTool({ name: 'my_app.employees.list' })` (your own tool) | Your app server | frame → host → Hub → (managed runtime: cluster, node) → your server, which then calls the Hub again for every read it needs |

The third path pays the trip to your server, the trip back, and one more Hub round trip for every
read your tool makes, one after another. In one production measurement, a screen that loaded
32 List rows through the app server took about 4 s; the same rows took about 0.2 s when the
browser read them from the Hub.

## 1. Read List data in the browser

Read and write List data from the UI with `app.rest()` (REST-first, see
[Auth & REST Integration](./auth-and-rest-integration.md)). The host makes the call as the
signed-in user and the Hub checks it against the scopes your app was granted.

| What the screen needs | Call | Scope |
|---|---|---|
| The lists of the room | `GET lists.listByRoomId?roomId=` | `lists:read` |
| A window of items, filtered and paged | `POST items.query` ([reference](./apis/items-query.md)) | `lists:query` |
| One item | `GET items.info?itemId=` | `lists:read` |
| Ranked search | `GET items.search` | `lists:read` |
| Create, update, move, delete items | `POST items.create`, `items.update`, `items.moveToStage`, `items.delete` | `lists:write` |

```tsx
const app = usePrivosApp();

const res = await app.rest({
  method: 'POST',
  path: 'items.query',
  body: { listId, filter: { archived: false }, count: 50, fields: ['name', 'stageId', 'customFields'] },
});
if (res.statusCode >= 400) throw new Error(`items.query answered ${res.statusCode}`);
const { items, nextCursor } = res.body;
```

Read the window the screen shows (`count` 20 to 50) and page on scroll with `nextCursor`; do not
read a whole list on open. Ask for the `fields` you render.

**The browser receives exactly what the user may see.** The Hub applies room membership and the
visibility of isolated lists to every read, so a user gets nothing through your app that they
could not open in the List view. The opposite also holds: if a field must stay hidden from a user
who can open the List, the browser is the wrong place to read it. Put that field in a list the user
cannot read (an isolated list, or a list in a room they are not in), or keep that one read on your
server and return a trimmed view.

**Keep on your server** only what the browser cannot do:

- work that needs a secret or a third-party API;
- a write that must enforce a business rule across several records, or must be idempotent across
  retries (pass an operation key);
- reads of data the user may not see, such as totals over other people's rows;
- background jobs, and tools that agents or other apps call.

Do not add a server tool that reads a List and returns it reshaped. Reshape it in the browser.

## 2. Draw the UI first, check in the background

The frame should show something useful on its first render, before any check finishes.

- **Render at once.** Draw the layout with the last saved copy (section 3) or a placeholder. Do not
  wait for the context push, an access check, a setup or status tool, a licence check or a version
  check before drawing; and never unmount a drawn screen while one of them refreshes.
- **Check in the background, in parallel.** Start every check when the frame mounts. A check that
  passes changes nothing on screen.
- **Open a dialog when the user must know or decide.** Missing access, setup that needs an owner,
  a failed save, a refresh that failed while an old copy is shown, a confirmation before a change
  that cannot be undone: each opens one modal dialog over the screen, with the action that resolves
  it (Retry, Confirm, the steps to get access). The screen stays visible behind it. Also show the
  degraded state inline, so it is still clear after the dialog is closed.
- **One dialog per problem**, not one per failed request.
- **Use an in-page `<dialog>`** opened with `showModal()`. `window.alert`, `confirm` and `prompt` do
  nothing in the app frame, because its sandbox does not allow modals.

`usePrivosCapability(scope)` reports `resolved` as soon as the host pushes the effective scopes.
While it is not resolved, show the saved copy and do not ask the Hub yet; when it resolves to not
granted, drop the saved copy and open the dialog.

## 3. Offline first

### In the UI

Show the copy the browser saved last, mark how old it is, refresh it in the background and replace
it when the fresh copy arrives.

- **Store the copy in `app.storage`.** The frame runs in an opaque origin, so its own
  `localStorage` and IndexedDB are not available. `app.storage` keeps values in the host's
  `localStorage` under `mcp-app:{appId}:`, so other apps cannot read them.
- **Key every copy by user and room**, with a version prefix you bump when the stored shape
  changes (`v1:{userId}:{roomId}:lists`). The storage belongs to the browser profile, not to the
  user, and the Hub does not clear it when the user logs out.
- **Store only what the screen shows**, and nothing you would not leave on a shared computer: no
  tokens, salaries, bank details or other personal data. The whole workspace page shares a few MB
  of `localStorage`, so keep each app to a few hundred KB, and treat a failed write as "no copy
  this time", never as an error for the user.
- **The copy is a display hint, never an authority.** Decide nothing from it that the Hub would
  refuse, and drop it when access is revoked.
- **Refresh** when the frame opens, when it becomes visible again (`visibilitychange`) and after
  your own writes. Do not poll in a tight loop: `items.query` allows 120 requests per minute per
  user.
- **Sync incrementally** with `items.query` and `filter.updatedAt.gte` set a little before the
  last sync time, to absorb clock skew, and merge by `_id`
  ([example](./apis/items-query.md#incremental-sync)). Deletions do not appear in that stream, and
  `items.deleted` is not yet open to apps through the bridge: re-read the window in full from time
  to time.
- **Write optimistically.** Show the change at once, send it, and on a refusal put the old value
  back and open a dialog. A write the network lost can wait in `app.storage` and be retried on the
  next open, provided it carries an operation key so a retry cannot apply it twice.

The app frame itself is served by the Hub, so an app cannot open while the workspace is
unreachable. Offline first here means that an open app never shows an empty screen because of a
slow or lost connection.

### In the app server

Each tool call should make as few Hub calls as it can, and none it made a moment ago.

- **Cache Hub reads that rarely change** in memory: room members, your installation or settings
  record, the ids of your lists. Use a TTL of 30 to 60 s and share one in-flight request between
  concurrent callers.
- **Weigh the TTL of authorization data.** A cached member list means a removed member keeps access
  until it expires. Keep that TTL short, or drop the entry on the room events your app receives.
- **Run independent Hub reads in parallel**, not one after another.
- **Answer a screen with one tool call**, not a chain of calls from the UI.

### In the data

- Shape a List so that everyone in its room may see every field in it. Put restricted fields in a
  separate list with narrower visibility.
- Keep stable ids, and let `_updatedAt` move on every change, so incremental sync works.

## Checklist

- [ ] The first render draws the screen with a saved copy or a placeholder; nothing waits for a check.
- [ ] List data is read with `app.rest()` from the UI, not through a tool of your own.
- [ ] Every check runs in the background; problems and confirmations open one `<dialog>`.
- [ ] Saved copies live in `app.storage`, keyed by version, user and room, with no secrets or
      personal data.
- [ ] Refresh happens on open, on visibility and after writes, never in a tight loop.
- [ ] The app server caches rarely changing Hub reads with a TTL and runs independent reads in
      parallel.
